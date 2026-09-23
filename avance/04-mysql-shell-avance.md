# TP 4 — Diagnostic complet via MySQL Shell et utilisation des commandes de recovery

**Module associé :** [Module 4 — Exploitation avancée de MySQL Shell](../modules/module-04-mysql-shell-avance.md)

## Exercice 1 — rescan() après une intervention manuelle

### 1.1 — Créer un écart entre la réalité GR et la metadata AdminAPI
Sur un membre, arrêtez Group Replication en SQL brut plutôt que via
l'AdminAPI :
```sql
STOP GROUP_REPLICATION;
```
Puis redémarrez-le manuellement, toujours en SQL brut (pas
`cluster.rejoinInstance()`) :
```sql
START GROUP_REPLICATION;
```

### 1.2 — Constater et corriger l'écart
```javascript
var cluster = dba.getCluster();
cluster.status();
```
> **Question :** la metadata AdminAPI reflète-t-elle immédiatement le
> membre comme `ONLINE`, ou observe-t-on un écart transitoire ?

```javascript
cluster.rescan();
```
> **Question :** que corrige concrètement `rescan()` ici ? Dans quel cas
> réel (hors TP) cette commande est-elle indispensable (indice : un membre
> ajouté/retiré directement en SQL, sans passer par l'AdminAPI, par un
> collègue peu familier des bonnes pratiques) ?

## Exercice 2 — Panne totale et reboot cluster

**Attention :** cet exercice coupe complètement le cluster. À réserver à un
environnement de TP, jamais en production sans fenêtre de maintenance.

### 2.1 — Provoquer la panne totale
```bash
# Sur les 3 nodes
sudo systemctl stop mysql
```
Depuis `router1` :
```javascript
var cluster = dba.getCluster();
cluster.status();
```
> **Question :** quel message obtenez-vous ? Le Shell peut-il encore
> interroger le cluster ?

### 2.2 — Identifier le membre à utiliser pour le reboot
Avant de redémarrer, il faut identifier quel membre a le GTID Set le plus
avancé (celui qui a le moins de risque de perte de données) :
```bash
# Redémarrer MySQL (pas Group Replication) sur chaque nœud, un par un,
# et comparer AVANT de lancer le reboot cluster
sudo systemctl start mysql
```
```sql
-- Sur chaque nœud, une fois MySQL (pas GR) redémarré
SELECT @@GLOBAL.gtid_executed;
```
> **Question :** comparez les 3 résultats avec `GTID_SUBSET`. Lequel des 3
> nœuds est le bon candidat pour lancer `rebootClusterFromCompleteOutage`
> (celui dont le GTID Set contient tous les autres) ?

### 2.3 — Reboot depuis le bon nœud
Depuis MySQL Shell, connecté au nœud identifié en 2.2 :
```javascript
shell.connect('clusteradmin:ClusterAdmin2026!@<IP_NOEUD_CHOISI>:3306');
var cluster = dba.rebootClusterFromCompleteOutage('trainingCluster');
cluster.status();
```
> **Question :** les 2 autres membres rejoignent-ils automatiquement, ou
> faut-il une action supplémentaire ?

## Corrections

<details>
<summary>Voir les corrections</summary>

1.1/1.2. Un écart transitoire peut apparaître (quelques secondes) avant que
   le cache metadata du Shell ne se resynchronise ; `rescan()` force cette
   resynchronisation immédiatement plutôt que d'attendre. Le cas réel
   typique : quelqu'un a ajouté/retiré un membre en `STOP/START
   GROUP_REPLICATION` direct un jour de panne, sans AdminAPI disponible
   (Shell non installé sur la machine d'astreinte, par exemple) — `rescan()`
   permet de "rattraper proprement" l'état après coup.

2.1. Le Shell renvoie une erreur indiquant qu'aucun membre n'est joignable
   / que le cluster est en panne totale — les méthodes habituelles
   (`getCluster()`, `status()`) échouent, seul `rebootClusterFromCompleteOutage()`
   s'applique dans cette situation.

2.2. Le nœud dont le `gtid_executed` **contient** ceux des deux autres
   (`GTID_SUBSET(gtid_des_autres, gtid_de_celui_ci)` = TRUE pour les deux
   comparaisons) est le bon candidat — démarrer le reboot depuis un nœud en
   retard entraînerait une perte des transactions que les autres avaient
   déjà.

2.3. Les 2 autres membres ne rejoignent pas automatiquement dans tous les
   cas : `rebootClusterFromCompleteOutage()` reconstruit le cluster autour
   du seul nœud depuis lequel la commande est lancée ; il faut ensuite
   ajouter (`cluster.addInstance()`) ou réintégrer (`rejoinInstance()`)
   explicitement les deux autres membres.

</details>
