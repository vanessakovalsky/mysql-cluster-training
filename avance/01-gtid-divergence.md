# TP 1 — Analyse de divergence GTID et validation d'une réintégration de nœud


**Prérequis :** cluster InnoDB opérationnel (3 nœuds).

## Exercice 1 — Provoquer et détecter une transaction errante

### 1.1 — Provoquer l'incident
Sur un SECONDARY (ex. `node3`), contourner volontairement Group
Replication :
```sql
SET GLOBAL super_read_only = 0;
SET GLOBAL read_only = 0;
CREATE DATABASE IF NOT EXISTS tp_errant;
CREATE TABLE tp_errant.t (id INT PRIMARY KEY);
INSERT INTO tp_errant.t VALUES (1);
SET GLOBAL super_read_only = 1;
```

### 1.2 — Détecter, sans savoir a priori où est le problème
Depuis chaque membre :
```sql
SELECT @@GLOBAL.gtid_executed;
```
Puis, sur le membre suspect :
```sql
SELECT GTID_SUBTRACT(@@GLOBAL.gtid_executed, RECEIVED_TRANSACTION_SET)
FROM performance_schema.replication_connection_status
WHERE channel_name = 'group_replication_applier';
```
> **Question :** quel GTID apparaît en résultat ? Confirmez qu'il
> correspond bien aux 2 transactions de l'exercice 1.1 (CREATE + INSERT).

### 1.3 — Évaluer le risque
> **Question :** que se passerait-il si ce nœud devait être **retiré puis
> réintégré** au cluster sans corriger cette transaction errante d'abord ?
> (Indice : `dba.checkInstanceState()` avant un `rejoinInstance()`.)

### 1.4 — Corriger
Deux options : purger la donnée errante manuellement (si elle n'a pas de
valeur), ou l'injecter dans les autres membres pour homogénéiser (rarement
souhaitable). Dans ce TP, purger :
```sql
SET GLOBAL super_read_only = 0;
DROP DATABASE tp_errant;
SET GLOBAL super_read_only = 1;
```
> **Question :** cette suppression elle-même génère-t-elle un nouveau GTID
> local, potentiellement errant à son tour ? Comment l'éviter en production
> (piste : `SET GTID_NEXT` pour rejouer sous un GTID déjà connu du groupe,
> ou repartir d'un clone complet si le doute persiste).

## Exercice 2 — Réintégration impossible par recovery, obligatoire par clone

### 2.1 — Simuler un retard trop important
Sur le PRIMARY, forcer une rotation et une purge de binlog agressive :
```sql
FLUSH BINARY LOGS;
-- Générer de l'activité pour avancer le GTID_EXECUTED du groupe
-- (boucle d'INSERT depuis router1, comme au Module 3 du support découverte)
PURGE BINARY LOGS BEFORE NOW();
```

### 2.2 — Retirer temporairement un membre puis tenter une recovery incrémentale
```javascript
var cluster = dba.getCluster();
cluster.removeInstance('clusteradmin:ClusterAdmin2026!@<IP_NODEX>:3306');
// ... laisser le cluster tourner un moment, générer de l'activité ...
cluster.addInstance('clusteradmin:ClusterAdmin2026!@<IP_NODEX>:3306',
  {recoveryMethod: 'incremental'});
```
> **Question :** l'ajout échoue-t-il ? Quel message d'erreur MySQL Shell
> renvoie-t-il concernant les GTID manquants ?

### 2.3 — Diagnostiquer avec GTID_SUBSET/SUBTRACT avant de corriger
```sql
-- Sur le PRIMARY
SELECT GTID_SUBSET('<gtid_purged_primary>', '<gtid_executed_nodeX>');
```
> **Question :** ce résultat confirme-t-il que la recovery incrémentale est
> impossible ? Pourquoi ?

### 2.4 — Corriger par clone
```javascript
cluster.addInstance('clusteradmin:ClusterAdmin2026!@<IP_NODEX>:3306',
  {recoveryMethod: 'clone'});
cluster.status();
```

## Corrections

<details>
<summary>Voir les corrections</summary>

1.2. Le GTID retourné correspond aux 2 transactions locales (CREATE
   DATABASE + CREATE TABLE + INSERT, selon le mode de commit — comptez les
   transactions réellement commitées). `RECEIVED_TRANSACTION_SET` ne les
   contient pas puisqu'elles n'ont jamais transité par le groupe.

1.3. `checkInstanceState()` renverrait un état signalant des transactions
   locales non reconnues par le cluster (`state: "error"` ou avertissement
   explicite selon la version). Un `rejoinInstance()` sur un nœud avec des
   transactions errantes non résolues est refusé ou nécessite un
   `recoveryMethod: "clone"` pour écraser l'état local incohérent.

1.4. Oui, un `DROP DATABASE` génère lui-même un nouveau GTID local sur ce
   membre, qui reste "à lui" mais ne pose pas de problème ici puisqu'il ne
   contredit aucune donnée existante ailleurs — c'est différent d'une
   transaction errante qui **modifie un état** divergent du reste du
   groupe. En production, le plus sûr en cas de doute reste un clone complet
   plutôt qu'une correction manuelle ligne par ligne.

2.2. L'ajout échoue avec une erreur indiquant que les GTID nécessaires à la
   recovery incrémentale ne sont plus disponibles dans les binlogs du
   donneur (transactions purgées).

2.3. `GTID_SUBSET` renvoie `FALSE` (ou 0) : le `gtid_purged` du PRIMARY
   n'est pas un sous-ensemble du `gtid_executed` du nœud candidat — il
   contient des GTID que ce nœud n'a jamais eus et qui ne sont plus
   disponibles en binlog pour les lui transmettre. La seule voie restante
   est un transfert physique complet (clone).

</details>
