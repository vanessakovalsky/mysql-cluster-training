# TP 4 — Créer l'InnoDB Cluster avec MySQL Shell


## Étape 1 — Se connecter et vérifier une instance

Depuis `router1` :
```bash
mysqlsh
```
```javascript
\js
shell.connect('clusteradmin@10.42.0.11:3306');
dba.configureInstance();
```
> `configureInstance()` sans argument analyse l'instance courante et
> **propose interactivement** les corrections nécessaires (variables GTID,
> comptes, redémarrage éventuel). C'est l'occasion de montrer aux
> apprenants ce qu'il vérifie exactement.

## Étape 2 — Répéter sur node2 et node3
```javascript
dba.configureInstance('clusteradmin:ClusterAdmin2026!@10.42.0.12:3306');
dba.configureInstance('clusteradmin:ClusterAdmin2026!@10.42.0.13:3306');
```

## Étape 3 — Créer le cluster depuis node1
```javascript
shell.connect('clusteradmin:ClusterAdmin2026!@10.42.0.11:3306');
var cluster = dba.createCluster('trainingCluster');
cluster.status();
```

## Étape 4 — Ajouter les 2 autres membres
```javascript
cluster.addInstance('clusteradmin:ClusterAdmin2026!@10.42.0.12:3306',
  {recoveryMethod: 'clone'});
cluster.addInstance('clusteradmin:ClusterAdmin2026!@10.42.0.13:3306',
  {recoveryMethod: 'clone'});
cluster.status();
```

## Étape 5 — Simuler une panne et observer le failover automatique
```bash
# Sur node1 (le PRIMARY courant)
sudo systemctl stop mysql
```
```javascript
// Depuis router1, quelques secondes plus tard
cluster.status();  // un nouveau PRIMARY doit être élu parmi node2/node3
```
> **Question :** combien de temps s'écoule entre l'arrêt de node1 et
> l'élection du nouveau PRIMARY ? Quelle variable contrôle ce délai
> (`group_replication_member_expel_timeout`) ?

## Étape 6 — Réintégrer node1
```bash
sudo systemctl start mysql
```
```javascript
cluster.status();  // node1 doit rejoindre en SECONDARY après recovery/clone
```

## Corrections / points d'attention

<details>
<summary>Voir les corrections</summary>

- Le délai par défaut avant expulsion d'un membre injoignable est piloté par
  `group_replication_member_expel_timeout` (5 secondes par défaut en 8.0
  récent). En dessous de ce délai, GR tolère une micro-coupure réseau sans
  reconfigurer le groupe.
- Après redémarrage, `node1` ne redevient **pas** automatiquement PRIMARY :
  en Single-Primary, un membre qui rejoint le groupe revient toujours en
  SECONDARY, sauf bascule explicite (`cluster.setPrimaryInstance()`). C'est
  un comportement volontaire pour éviter les bascules intempestives.
- Si `node1` a raté trop de transactions pendant son arrêt, GR choisira
  automatiquement un rattrapage par **clone** plutôt que par recovery
  incrémentale (Module 3.6/3.14) — visible dans les logs MySQL Shell en
  `DEBUG3`.

</details>
