# TP 10 — Simulation de promotion et validation du fencing

**Module associé :** [Module 10 — Fencing et bascule inter-cluster](../modules/module-10-fencing-bascule.md)

**Prérequis :** ClusterSet créé (TP9).

## Exercice 1 — Valider le fencing avant toute bascule

Sur un membre du cluster REPLICA :
```sql
SELECT @@GLOBAL.super_read_only;
INSERT INTO tp_gtid.compteur VALUES (9999, 1);
```
> **Question :** l'`INSERT` échoue-t-il ? Confirmez que c'est le cas même
> en vous connectant en tant qu'utilisateur `clusteradmin` (compte
> disposant normalement de tous les privilèges) — le fencing s'applique-t-il
> indépendamment du compte utilisé ?

## Exercice 2 — Bascule planifiée (site PRIMARY toujours disponible)

```javascript
var clusterSet = dba.getClusterSet();
clusterSet.status({extended: 1});

clusterSet.setPrimaryCluster('siteB-cluster');
clusterSet.status();
```
> **Question :** l'ancien cluster PRIMARY (site A) devient-il
> automatiquement fencé (REPLICA) après cette commande ? Vérifiez
> `super_read_only` sur un de ses membres.

Sur l'ancien PRIMARY (maintenant REPLICA) :
```sql
INSERT INTO tp_gtid.compteur VALUES (10000, 1);
```
> **Question :** cette écriture échoue-t-elle désormais, alors qu'elle
> aurait réussi avant la bascule ?

## Exercice 3 — Bascule d'urgence et tags GTID

### 3.1 — Simuler une panne totale du PRIMARY actuel (site B après l'exercice 2)
```bash
# Sur tous les membres du cluster actuellement PRIMARY
sudo systemctl stop mysql
```

### 3.2 — Forcer la promotion du site A
```javascript
clusterSet.forcePrimaryCluster('trainingCluster');   // site A, ancien PRIMARY d'origine
clusterSet.status();
```

### 3.3 — Observer le nouveau tag GTID
Une fois quelques transactions générées sur ce nouveau PRIMARY :
```sql
SHOW VARIABLES LIKE 'gtid_executed';
```
> **Question :** repérez, dans le GTID Set, la présence d'un tag (format
> `source_id:tag:transaction_id`, Module 1/programme découverte). Ce tag
> est-il différent de celui (absent ou différent) qu'avait ce même cluster
> avant l'incident ?

### 3.4 — Réintégrer l'ancien PRIMARY (site B) une fois revenu
```bash
# Sur les membres du site B
sudo systemctl start mysql
```
```javascript
clusterSet.rejoinCluster('siteB-cluster');
clusterSet.status({extended: 1});
```
> **Question :** cette réintégration nécessite-t-elle un clone complet ou
> un rejeu incrémental ? Sur quoi cela dépend-il (indice : présence ou
> absence de transactions divergentes générées côté site B après sa
> "perte" et avant son redémarrage — dans ce TP, aucune écriture directe
> n'a eu lieu pendant la coupure, donc...).

## Corrections

<details>
<summary>Voir les corrections</summary>

1. Oui, l'`INSERT` échoue même en `clusteradmin` : le fencing via
   `super_read_only` s'applique au niveau serveur, indépendamment des
   privilèges du compte utilisé — c'est une protection structurelle, pas
   une restriction applicative contournable par un compte "admin".
2. Oui, l'ancien PRIMARY devient automatiquement fencé
   (`super_read_only = 1`) dès que `setPrimaryCluster()` termine son
   exécution — la bascule planifiée gère les deux côtés en une seule
   opération cohérente.
   L'écriture testée ensuite échoue bien, pour la même raison qu'à
   l'exercice 1.
3.3. Un nouveau tag apparaît (ou change), distinct de celui utilisé avant
   l'incident — c'est le mécanisme qui permet de distinguer sans ambiguïté
   les transactions post-promotion de celles antérieures, même si elles
   pouvaient techniquement porter des numéros de séquence qui se
   recoupent.
3.4. Sans transaction divergente générée côté site B pendant la coupure
   (aucune écriture directe n'a eu lieu, contrairement à un scénario
   d'errant transaction), un rejeu incrémental est possible en théorie ;
   en pratique, `rejoinCluster()` choisit très souvent un clone complet
   par sécurité par défaut, quitte à être plus long qu'un strict
   nécessaire — comportement déjà observé dans le support découverte
   (Module ClusterSet).

</details>
