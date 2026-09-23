# TP 6 — Mettre en place un ClusterSet (DR multi-site simulé)

**Module associé :** [Module 6 — InnoDB ClusterSet](../modules/module-06-clusterset.md)

> Pour ce TP, on simule 2 « sites » avec les 3 nœuds déjà provisionnés :
> `node1+node2` = Site A (`trainingCluster`, déjà créé au Module 4), et on
>  l'on redimensionne temporairement
> `node3` en cluster REPLICA à un seul nœud pour illustrer le principe (en
> conditions réelles de production, chaque site aurait ses propres 3 nœuds).

## Étape 1 — Créer le ClusterSet à partir du cluster existant
```javascript
// Depuis mysqlsh, connecté au PRIMARY du cluster existant
shell.connect('clusteradmin:ClusterAdmin2026!@10.42.0.11:3306');
var cluster = dba.getCluster();
var clusterSet = cluster.createClusterSet('productionClusterSet');
clusterSet.status();
```

## Étape 2 — Ajouter un cluster REPLICA (site distant simulé)
```javascript
// Sur l'instance destinée au site B (ex: node3 reconfiguré, ou 4e instance)
dba.configureInstance('clusteradmin:ClusterAdmin2026!@10.42.0.13:3306');

// Depuis la connexion au ClusterSet
var drCluster = clusterSet.createReplicaCluster(
  'clusteradmin:ClusterAdmin2026!@10.42.0.13:3306',
  'trainingCluster-dr',
  {recoveryMethod: 'clone'}
);
clusterSet.status({extended: 1});
```

## Étape 3 — Vérifier le fencing
```sql
-- Sur l'instance du cluster REPLICA
SELECT @@GLOBAL.super_read_only;   -- doit être = 1 (ON)
```
```javascript
// Tenter une écriture directe sur le REPLICA (doit échouer)
```
```sql
INSERT INTO tp_gtid.compteur VALUES (999, 1);
-- Erreur attendue : --read-only (super_read_only)
```

## Étape 4 — Simuler un failover de site et une réintégration
```javascript
// Simuler l'indisponibilité du site A (couper node1/node2, ou juste observer)
clusterSet.status({extended: 1});

// Forcer la promotion du cluster REPLICA en PRIMARY
clusterSet.forcePrimaryCluster('trainingCluster-dr');
clusterSet.status();

// Une fois le site A revenu :
clusterSet.rejoinCluster('trainingCluster');
clusterSet.status({extended: 1});
```

> **Question :** après `rejoinCluster()`, `trainingCluster` (site A)
> a-t-il été rembobiné (clone complet) ou a-t-il pu rejouer uniquement un
> delta ? Regardez les logs MySQL Shell en `--log-level=DEBUG3` pour la
> réponse exacte dans votre scénario de test.

## Corrections / points clés

<details>
<summary>Voir les corrections</summary>

- `super_read_only=1` sur le REPLICA est la manifestation concrète du
  fencing (6.2) : même sans coupure réseau, aucune écriture directe n'est
  jamais possible sur un cluster REPLICA, garantissant qu'il ne peut pas
  diverger de sa propre initiative.
- Dans la grande majorité des scénarios de TP (peu de transactions
  générées entre la coupure et la réintégration), `rejoinCluster()` choisit
  un **clone complet** par sécurité plutôt qu'un rejeu incrémental — c'est
  le comportement par défaut le plus sûr, quitte à être plus long qu'un
  simple rattrapage GTID.
- En production, la fréquence de ce type d'incident justifie de toujours
  budgétiser le temps de clonage complet dans son plan de reprise
  d'activité (RTO), plutôt que de supposer un rejeu incrémental rapide.

</details>
