# TP 8 — Simulation d'un clone échoué, analyse et correction

**Module associé :** [Module 8 — Clonage et recovery avancé](../modules/module-08-clonage-recovery-avance.md)

## Préparation

Retirez `node3` du cluster pour cet exercice (comme au TP2/TP4) :
```javascript
var cluster = dba.getCluster();
cluster.removeInstance('clusteradmin:ClusterAdmin2026!@<IP_NODE3>:3306');
```

## Exercice 1 — Échec immédiat : donneur non autorisé

Sur `node3` :
```sql
INSTALL PLUGIN clone SONAME 'mysql_clone.so';
-- Volontairement OMIS : SET GLOBAL clone_valid_donor_list
CLONE INSTANCE FROM 'clusteradmin'@'<IP_NODE1>':3306
  IDENTIFIED BY 'ClusterAdmin2026!';
```
> **Question :** quel message d'erreur obtenez-vous ? À quel stade
> (avant tout transfert de données, ou en cours de transfert) ?

Corrigez :
```sql
SET GLOBAL clone_valid_donor_list = '<IP_NODE1>:3306';
```

## Exercice 2 — Échec en cours de transfert : espace disque insuffisant (simulé)

> Sur une vraie instance de TP, remplir complètement le disque est risqué
> pour le reste du système. Simulez plutôt en réduisant temporairement le
> quota disponible via un fichier de bourrage contrôlé, si votre volume
> EBS le permet, ou traitez cet exercice en lecture de log seulement à
> partir du corrigé si l'environnement ne s'y prête pas.

```bash
# Sur node3, remplir une bonne partie de l'espace libre restant
df -h /var/lib/mysql
fallocate -l <TAILLE_PROCHE_DE_L_ESPACE_LIBRE> /var/lib/mysql/filler.tmp
```
Relancez le clone :
```sql
CLONE INSTANCE FROM 'clusteradmin'@'<IP_NODE1>':3306
  IDENTIFIED BY 'ClusterAdmin2026!';
```
Pendant l'opération (ou juste après l'échec), sur `node3` :
```sql
SELECT stage, state, error_no, error_message
FROM performance_schema.clone_progress
WHERE error_no != 0;
```
> **Question :** à quelle étape (`FILE COPY`, `PAGE COPY`, `REDO COPY`)
> l'échec survient-il ? Le message d'erreur est-il explicite sur la cause
> (espace disque) ?

Nettoyez :
```bash
rm -f /var/lib/mysql/filler.tmp
```

## Exercice 3 — Clone réussi et réintégration

```sql
CLONE INSTANCE FROM 'clusteradmin'@'<IP_NODE1>':3306
  IDENTIFIED BY 'ClusterAdmin2026!';
```
Une fois le service redémarré automatiquement après le clone :
```javascript
cluster.addInstance('clusteradmin:ClusterAdmin2026!@<IP_NODE3>:3306',
  {recoveryMethod: 'incremental'});
cluster.status();
```
> **Question :** pourquoi `incremental` est correct ici plutôt que
> `clone` (déjà vu au TP3, mais reformulez avec vos propres mots) ?

## Corrections

<details>
<summary>Voir les corrections</summary>

1. L'erreur survient **avant tout transfert** : MySQL valide la présence
   du donneur dans `clone_valid_donor_list` avant même d'établir la
   connexion de clonage. C'est une validation de configuration, pas un
   problème de transfert.
2. L'échec survient typiquement pendant `FILE COPY` (la phase de copie
   physique des fichiers, la plus consommatrice d'espace disque) ; le
   message d'erreur mentionne généralement explicitement un problème
   d'espace disque (`No space left on device` ou équivalent MySQL),
   directement exploitable sans investigation supplémentaire.
3. Les données de `node3` sont déjà à jour grâce au clone manuel réussi de
   l'exercice 3 : `addInstance` n'a besoin de rattraper que le petit delta
   de transactions survenu depuis la fin du clone, via recovery
   incrémentale classique — reforcer un `clone` referait tout le transfert
   physique inutilement.

</details>
