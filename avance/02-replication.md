# TP 2 — Observation des flux de réplication et analyse de la certification


## Exercice 1 — Suivre un cycle de transaction de bout en bout

1. Ouvrez une session sur le noeud 1 `performance_schema.replication_group_member_stats`
   via une boucle `watch` shell, rafraîchie
   toutes les secondes, sur le PRIMARY.
```
watch -n 1 'mysql -u <utilisateur> -p"<mot_de_passe>" -e "SELECT * FROM performance_schema.replication_group_member_stats\G"'
```
2. Créer la structure de la base et de la table (dans un autre terminal):
```
CREATE DATABASE IF NOT EXISTS tp_gtid;

USE tp_gtid;

CREATE TABLE IF NOT EXISTS compteur (
    id INT PRIMARY KEY,
    valeur INT
);
```
3. Depuis un autre terminal, insérez une seule ligne :
   ```sql
   INSERT INTO tp_gtid.compteur VALUES (5000, 1);
   ```
4. Observez l'évolution de `COUNT_TRANSACTIONS_CHECKED` avant/après.

> **Question :** de combien ce compteur augmente-t-il pour une seule
> transaction applicative ? Est-ce cohérent avec le nombre de membres du
> cluster (chaque membre certifie-t-il indépendamment) ?

## Exercice 2 — Provoquer un conflit de certification et lire les métriques

1. Basculez en Multi-Primary : `cluster.switchToMultiPrimaryMode()`.
2. Sur `node2` ET `node3` simultanément :
   ```sql
   START TRANSACTION;
   UPDATE tp_gtid.compteur SET valeur = valeur + 1 WHERE id = 5000;
   COMMIT;
   ```
3. Sur les deux nœuds :
   ```sql
   SELECT COUNT_TRANSACTIONS_CHECKED, COUNT_CONFLICTS_DETECTED,
          COUNT_TRANSACTIONS_LOCAL_ROLLBACK
   FROM performance_schema.replication_group_member_stats
   WHERE MEMBER_ID = @@server_uuid;
   ```
> **Question :** `COUNT_CONFLICTS_DETECTED` augmente-t-il sur les deux
> nœuds ou un seul ? Expliquez à partir du mécanisme de certification
> (Module 2.3) pourquoi c'est le membre qui "arrive en second" dans l'ordre
> total GCS qui subit le rollback, et non un membre décidé à l'avance.

## Exercice 3 — Diagnostiquer un canal recovery qui ne se termine jamais

**Scénario simulé :** un nœud reste bloqué en recovery à cause d'un
problème réseau intermittent avec le donneur.

1. Retirez un membre : `cluster.removeInstance(...)`.
2. Sur ce membre, avant de le réintégrer, limitez artificiellement la bande
   passante réseau vers les autres nœuds
   Trouve le nom de ton interface réseau (ex: eth0, ens33, enp0s3) :
   ```
   ip a
   ```
   Pour brider le trafic sortant de l'interface (ex: eth0) à 100 kbit/s (très lent, idéal pour observer les files d'attente/lag) :
   ```
   sudo tc qdisc add dev eth0 root tbf rate 100kbit burst 32kbit latency 400ms
   ```
   
4. Relancez `cluster.addInstance(..., {recoveryMethod: 'incremental'})`.
5. Pendant l'opération, sur le nœud candidat :
   ```sql
   SELECT channel_name, service_state
   FROM performance_schema.replication_connection_status
   WHERE channel_name = 'group_replication_recovery';
   ```
> **Question :** si le canal recovery reste à `ON` largement au-delà du
> temps normalement nécessaire pour rattraper le retard généré, quelle
> commande MySQL Shell permet de forcer un nouveau donneur
> (`group_replication_recovery` peut retenter avec un autre membre) ? Et
> quelle variable système contrôle le nombre de tentatives
> (`group_replication_recovery_retry_count`) ?

## Corrections

<details>
<summary>Voir les corrections</summary>

1. Le compteur augmente de 1 **sur chaque membre** (chaque nœud certifie
   indépendamment sa copie de la même transaction diffusée par GCS) — donc
   d'autant que le nombre de membres du cluster, pas une seule fois de
   façon centralisée.
2. `COUNT_CONFLICTS_DETECTED` augmente uniquement sur le membre dont la
   transaction arrive **en second** dans l'ordre total établi par GCS/Paxos
   — cet ordre dépend de la diffusion réseau réelle au moment T, pas d'une
   règle déterministe connue à l'avance (ex. pas "toujours node2 gagne").
   C'est pour cette raison que le Multi-Primary exige une application
   capable de rejouer une transaction rejetée, sans supposer quel nœud
   "gagnera" par avance.
3. `group_replication_recovery_retry_count` (défaut 10) borne le nombre de
   tentatives de connexion à un donneur avant abandon. Si le canal reste
   bloqué, on peut aussi forcer manuellement en retirant puis réintégrant
   le membre avec `recoveryMethod: "clone"`, qui change complètement de
   mécanisme plutôt que de retenter la même recovery incrémentale.

</details>
