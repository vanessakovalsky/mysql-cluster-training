# TP — Explorer l'architecture interne

**Prérequis :** cluster créé (voir Module 4, script `02-create-cluster.js`)
— si ce TP est fait avant le Module 4, utiliser un Group Replication monté
manuellement en SQL, ou anticiper la création du cluster.

### Exercice 1 — Observer le rôle des canaux
```sql
SELECT channel_name, service_state, count_transactions_in_queue,
       count_transactions_applied
FROM performance_schema.replication_applier_status_by_coordinator;
```
> **Question :** identifiez, sur le PRIMARY, si le canal recovery est actif.
> Pourquoi ne devrait-il pas l'être en fonctionnement normal ?

### Exercice 2 — Provoquer et observer un conflit en Multi-Primary
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

### Exercice 3 — Observer le flow control sous charge
```bash
# Depuis router1 ou un nœud, générer de la charge en écriture (ex: sysbench ou boucle de INSERT)
for i in $(seq 1 5000); do
  mysql -h<IP_NODE1> -uclusteradmin -p'ClusterAdmin2026!' \
    -e "INSERT INTO tp_gtid.compteur VALUES ($((100+i)), $i);"
done
```
Pendant l'exécution, sur un autre terminal :
```sql
SELECT * FROM performance_schema.replication_group_member_stats;
```
> **Question :** observez `COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE`. Que
> se passe-t-il si vous réduisez artificiellement
> `group_replication_flow_control_applier_threshold` à une petite valeur (ex.
> 10) avant de relancer le test ?

### Corrections

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
3. `COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE` augmente lorsque le débit
   d'écriture dépasse la capacité d'application d'un membre. En réduisant le
   seuil de flow control, le throttling se déclenche plus tôt : les
   `INSERT` sur les autres nœuds ralentissent volontairement, ce qui limite
   la croissance de la queue au prix d'un débit global plus faible — un
   compromis explicite cohérence/latence vs débit.

</details>
