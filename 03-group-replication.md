# TP 3 — Explorer l'architecture interne de Group Replication

**Prérequis :** le cluster InnoDB doit déjà exister (créé au Module 4, TP
"Créer l'InnoDB Cluster avec MySQL Shell"). Dans le déroulé pédagogique,
la **théorie** du Module 3 est vue avant le Module 4, mais ce TP, lui,
s'exécute après la création effective du cluster — pensez à l'intercaler
en pratique après le TP du Module 4, même si le module théorique est
numéroté avant. Voir la remarque de planning en fin de ce fichier.

## Exercice 1 — Observer le rôle des canaux
```sql
SELECT channel_name, service_state, count_transactions_in_queue,
       count_transactions_applied
FROM performance_schema.replication_applier_status_by_coordinator;
```
> **Question :** identifiez, sur le PRIMARY, si le canal recovery est actif.
> Pourquoi ne devrait-il pas l'être en fonctionnement normal ?

## Exercice 2 — Provoquer et observer un conflit en Multi-Primary
1. Basculez le cluster en Multi-Primary (`cluster.switchToMultiPrimaryMode()`
   depuis MySQL Shell).
2. Ouvrez 2 sessions SQL simultanées, une sur `node2`, une sur `node3`.
3. Sur chaque session, en même temps :
   ```sql
   START TRANSACTION;
   UPDATE tp_gtid.compteur SET valeur = valeur + 1 WHERE id = 1;
   COMMIT;
   ```
> **Question :** l'une des deux transactions échoue-t-elle ? Avec quel
> message d'erreur ? Relancez l'observation sur
> `performance_schema.replication_group_member_stats` (colonne
> `COUNT_TRANSACTIONS_ROLLBACK`).

## Exercice 3 — Tuning du flow control avec mesure d'impact

**Objectif :** ne pas se contenter d'observer le flow control, mais
**mesurer chiffrément** son effet sur le débit d'écriture — ce qui est la
vraie question qu'un DBA se pose en production (quel compromis
cohérence/latence vs débit ?).

### 3.1 — Mesure de référence (flow control par défaut)

Depuis `router1` (ou un nœud), chronométrez une charge d'écriture :
```bash
time (for i in $(seq 1 3000); do
  mysql -h<IP_NODE1> -uclusteradmin -p'ClusterAdmin2026!' \
    -e "INSERT INTO tp_gtid.compteur VALUES ($((1000+i)), $i);" 2>/dev/null
done)
```
Notez le temps total affiché par `time` (ligne `real`).

Pendant l'exécution (dans un autre terminal), observez la file d'attente
d'application sur les SECONDARY :
```sql
SELECT MEMBER_ID, COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE
FROM performance_schema.replication_group_member_stats;
```

### 3.2 — Réduire drastiquement le seuil de flow control

```sql
-- Sur le PRIMARY
SET GLOBAL group_replication_flow_control_applier_threshold = 10;
SET GLOBAL group_replication_flow_control_mode = 'QUOTA';
```

Nettoyez la table (`DELETE FROM tp_gtid.compteur WHERE id > 1000;`) et
relancez **exactement la même mesure** qu'en 3.1 (même boucle, même nombre
d'itérations).

> **Question :** comparez les deux temps `real`. Le débit a-t-il
> significativement baissé ? Expliquez pourquoi, en vous appuyant sur ce que
> vous avez observé sur `COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE` dans
> les deux cas.

### 3.3 — Restaurer la configuration par défaut

```sql
SET GLOBAL group_replication_flow_control_applier_threshold = 25000;
```
(25000 est la valeur par défaut MySQL 8.0 — vérifiable avant modification
avec `SHOW VARIABLES LIKE 'group_replication_flow_control_applier_threshold';`
si vous voulez restaurer la valeur exacte d'origine plutôt qu'une valeur
générique.)

## Exercice 4 — Clonage manuel guidé (reconstruire un membre)

**Objectif :** vivre, étape par étape, ce que fait `cluster.addInstance(...,
{recoveryMethod: "clone"})` en coulisses (Module 3.14) — en reconstruisant
réellement `node3` par clonage physique depuis `node1`.

> **Attention :** cet exercice retire temporairement `node3` du cluster et
> efface ses données. Le cluster tolère cette perte (il reste 2 nœuds sur
> 3, donc toujours disponible en écriture), mais prévenez les apprenants
> qu'il ne faut pas faire cet exercice sur un cluster de production.

### 4.1 — Retirer node3 du cluster

Depuis MySQL Shell (`router1`) :
```javascript
var cluster = dba.getCluster();
cluster.removeInstance('clusteradmin:ClusterAdmin2026!@<IP_NODE3>:3306');
cluster.status();
```
> **Question :** combien de membres le cluster affiche-t-il maintenant ?
> Le cluster est-il toujours disponible en écriture ?

### 4.2 — Simuler une perte de données sur node3

Sur `node3` directement :
```sql
STOP GROUP_REPLICATION;   -- si encore actif malgré le removeInstance
SET GLOBAL super_read_only = 0;
DROP DATABASE IF EXISTS tp_gtid;
```

### 4.3 — Cloner manuellement depuis node1

Sur `node3` :
```sql
INSTALL PLUGIN clone SONAME 'mysql_clone.so';
SET GLOBAL clone_valid_donor_list = '<IP_PRIVEE_NODE1>:3306';
CLONE INSTANCE FROM 'clusteradmin'@'<IP_PRIVEE_NODE1>':3306
  IDENTIFIED BY 'ClusterAdmin2026!';
```
> Le processus MySQL de `node3` redémarre automatiquement à la fin du
> clonage — la session SSH/mysql se coupe, c'est normal.

Une fois reconnecté, vérifiez que les données sont bien revenues :
```sql
SELECT * FROM tp_gtid.compteur LIMIT 5;
SHOW VARIABLES LIKE 'server_uuid';
```
> **Question :** comparez ce `server_uuid` à celui que `node3` avait avant
> le clonage (si vous l'aviez noté au Module 2). A-t-il changé ? Pourquoi
> est-ce indispensable (voir Module 2.1) ?

### 4.4 — Réintégrer node3 dans le cluster

Depuis MySQL Shell :
```javascript
cluster.addInstance('clusteradmin:ClusterAdmin2026!@<IP_NODE3>:3306',
  {recoveryMethod: 'incremental'});
cluster.status();
```
> **Question :** pourquoi peut-on utiliser `recoveryMethod: "incremental"`
> ici plutôt que `"clone"`, alors qu'on vient justement de faire un clonage
> manuel ? (Réponse : les données sont déjà à jour depuis le clonage
> manuel — seule la recovery du delta minime généré depuis est nécessaire,
> Group Replication n'a pas besoin de recloner ce qui vient de l'être.)

## Corrections

<details>
<summary>Voir les corrections</summary>

1. Le canal `group_replication_recovery` ne doit être actif que
   **temporairement**, lorsqu'un membre rejoint le groupe. S'il reste actif
   en permanence sur un nœud ONLINE, c'est le signe d'un problème de
   configuration ou d'un nœud qui boucle en recovery sans jamais atteindre
   l'état ONLINE.
2. Oui : la transaction arrivée en second dans l'ordre total de
   certification échoue avec une erreur de type `ER_TRANSACTION_ROLLBACK_
   DURING_COMMIT` (conflit de certification). Le compteur
   `COUNT_TRANSACTIONS_ROLLBACK` augmente de 1.
3. Avec un seuil de flow control très bas (10), le débit d'écriture baisse
   nettement : dès que la file d'attente d'un SECONDARY dépasse 10
   transactions non appliquées, le PRIMARY est volontairement ralenti pour
   laisser le temps au SECONDARY de rattraper. Le temps `real` de la
   deuxième mesure doit être visiblement supérieur à la première — c'est le
   compromis cohérence (aucun membre ne prend un retard non borné) contre
   débit (le groupe entier est bridé par le membre le plus lent).
4.1. Le cluster passe à 2 membres, toujours disponible en écriture (il
   reste une majorité pour le consensus avec 2 nœuds sur les 3 d'origine).
4.3. Oui, le `server_uuid` a changé après clonage : c'est le clone plugin
   qui régénère `auto.cnf` automatiquement (Module 2.1), précisément pour
   éviter qu'un serveur reconstruit ne partage l'identité GTID d'un autre
   membre du groupe, ce qui corromprait la certification.
4.4. Les données de `node3` sont déjà à jour grâce au clonage manuel de
   l'exercice précédent : `addInstance` n'a donc besoin de rattraper que le
   petit delta de transactions survenues depuis (recovery incrémentale via
   GTID), pas de tout recloner depuis zéro.

</details>

## Note de planning

Le TP3 nécessite un cluster déjà créé (Module 4). Si votre déroulé suit
l'ordre des modules à la lettre (Module 3 avant Module 4), reportez ce TP
après le TP de création du cluster, ou anticipez une création de cluster
minimale en début de Module 3 pour pouvoir illustrer les concepts au fur et
à mesure de la théorie plutôt qu'en bloc différé.
