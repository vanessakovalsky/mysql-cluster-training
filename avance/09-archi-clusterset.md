# TP 9 — Analyse d'une architecture ClusterSet

**Module associé :** [Module 9 — Architecture de MySQL InnoDB ClusterSet](../modules/module-09-clusterset-architecture.md)

## Partie 1 — Mise en place (si pas déjà fait dans une session précédente)

```javascript
shell.connect('clusteradmin:ClusterAdmin2026!@<IP_NODE1>:3306');
var cluster = dba.getCluster();
var clusterSet = cluster.createClusterSet('productionClusterSet');

dba.configureInstance('clusteradmin:ClusterAdmin2026!@<IP_SITE_B>:3306');
var drCluster = clusterSet.createReplicaCluster(
  'clusteradmin:ClusterAdmin2026!@<IP_SITE_B>:3306',
  'siteB-cluster',
  {recoveryMethod: 'clone'}
);
```

## Partie 2 — Observer le canal de réplication inter-cluster

Sur le membre "receveur" du cluster REPLICA :
```sql
SELECT channel_name, service_state
FROM performance_schema.replication_connection_status
WHERE channel_name = 'clusterset_replication';
```
Générez de l'activité sur le PRIMARY, puis mesurez le lag :
```sql
-- Sur le PRIMARY, noter l'heure et insérer
INSERT INTO tp_gtid.compteur VALUES (9000, NOW());

-- Sur le REPLICA, quelques secondes après
SELECT * FROM tp_gtid.compteur WHERE id = 9000;
```
> **Question :** quel est le délai observé entre l'écriture sur le
> PRIMARY et sa visibilité sur le REPLICA ? Ce délai est-il comparable au
> lag intra-cluster observé aux TP précédents (Group Replication,
> quelques millisecondes à quelques secondes en charge normale) ?

## Partie 3 — Étude de cas (analyse, sans manipulation)

**Contexte fourni :** votre entreprise a 2 datacenters, Paris (PRIMARY) et
Marseille (REPLICA), reliés par une liaison WAN avec une latence moyenne de
40 ms et une bande passante garantie de 100 Mbps. Le PRIMARY produit en
moyenne 2000 transactions/seconde de 500 octets chacune en pointe.

> **Question 1 :** la bande passante WAN est-elle suffisante pour ce débit
> en régime soutenu ? (Calcul : 2000 × 500 octets = 1 Mo/s ≈ 8 Mbps — à
> comparer aux 100 Mbps disponibles, marge confortable a priori, mais que
> se passe-t-il si un pic ponctuel dépasse largement cette moyenne ?)

> **Question 2 :** si la liaison WAN tombe pendant 10 minutes en pleine
> activité, quel est l'ordre de grandeur du volume de transactions
> accumulées côté PRIMARY, non encore répliquées vers Marseille ? Quel est
> l'impact sur le RPO si une bascule d'urgence (Module 10) devait être
> décidée pendant cette coupure ?

> **Question 3 :** pourquoi cette architecture ne protège-t-elle **pas**
> contre une panne applicative ou une corruption de données introduite par
> l'application elle-même (contrairement à une panne d'infrastructure) ?
> Quelle brique complémentaire (hors périmètre MySQL) faudrait-il pour
> couvrir ce risque ?

## Corrections

<details>
<summary>Voir les corrections</summary>

Partie 2. Le lag inter-cluster observé dépend fortement de la latence WAN
   réelle entre les sites (contrairement au lag intra-cluster, borné par le
   réseau local) — attendez-vous à un ordre de grandeur de quelques
   centaines de millisecondes à quelques secondes selon la distance
   géographique simulée entre vos instances AWS, sensiblement plus élevé
   qu'un lag Group Replication intra-site en charge normale.

Q1. Oui largement en moyenne (8 Mbps sur 100 Mbps disponibles), mais un pic
   ponctuel à, disons, 10x la moyenne (20 000 tps) saturerait la liaison
   (80 Mbps, proche de la limite avec d'autres flux WAN concurrents) — la
   moyenne seule ne suffit pas à dimensionner, il faut aussi la variance/
   les pics.

Q2. À 2000 tps × 500 octets, 10 minutes de coupure représentent environ
   1,2 million de transactions (2000 × 600s) non répliquées, soit
   potentiellement plusieurs centaines de Mo de données côté PRIMARY que
   Marseille n'aurait jamais reçues. En cas de bascule d'urgence vers
   Marseille pendant cette fenêtre, c'est le volume de données
   **définitivement perdu** (RPO) si Paris ne peut pas être récupéré.

Q3. ClusterSet protège contre une panne d'infrastructure (matériel, réseau,
   datacenter) en répliquant fidèlement — mais si l'application écrit une
   donnée corrompue ou erronée sur le PRIMARY, cette corruption est
   répliquée **exactement de la même façon** vers le REPLICA (la
   réplication ne fait pas de distinction entre bonne et mauvaise donnée).
   Une protection contre ce risque nécessite une solution de sauvegarde
   avec rétention/point-in-time recovery (ex. `util.dumpInstance()` +
   binlogs conservés, ou une solution de backup tierce), indépendante de
   ClusterSet.

</details>
