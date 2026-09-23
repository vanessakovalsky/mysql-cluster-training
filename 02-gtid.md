# TP 2 — Manipulation des GTID

**Module associé :** [Module 2 — GUID & GTID](../modules/module-02-guid-gtid.md)

**Prérequis :** environnement du Module 0 provisionné, connexion SSH sur
`node1`, `node2`, `node3`. Ce TP se déroule **avant** la création du cluster
(Module 4) : les nœuds sont encore des serveurs MySQL indépendants, ce qui
est volontaire — cela permet de manipuler la réplication GTID "à la main"
avant de voir Group Replication l'automatiser au Module 3/4.

## Exercice 1 — Observer la génération de GTID

Sur `node1` :
```sql
mysql -uroot -p'FormationMySQL2026!'
CREATE DATABASE IF NOT EXISTS tp_gtid;
USE tp_gtid;
CREATE TABLE compteur (id INT PRIMARY KEY, valeur INT);
INSERT INTO compteur VALUES (1, 100);
SHOW VARIABLES LIKE 'gtid_executed';
```
> **Question :** notez le GTID Set obtenu. À quoi correspond le dernier
> numéro de la plage ?

## Exercice 2 — Comparer GTID_EXECUTED entre 2 serveurs non répliqués

Exécutez la même requête sur `node2` (qui n'est pas encore répliqué à ce
stade). Comparez les deux `server_uuid` dans les GTID obtenus.
> **Question :** pourquoi les deux GTID Sets utilisent-ils un `source_id`
> différent bien que la structure des données soit identique ?

## Exercice 3 — GTID_SUBTRACT en pratique

En utilisant les deux GTID Sets récupérés à l'exercice 2, exécutez sur
`node1` :
```sql
SELECT GTID_SUBTRACT(@@GLOBAL.gtid_executed, '<GTID_SET_DE_NODE2>');
```
> **Question :** que représente le résultat ? Est-ce cohérent avec le fait
> que les deux serveurs n'ont jamais été synchronisés ?

## Exercice 4 — Construire, casser puis réparer une réplication GTID manuelle

**Objectif :** manipuler pour de vrai ce que Group Replication automatisera
au Module 3/4 — mettre en place une réplication basée GTID, provoquer un
incident réaliste, le diagnostiquer avec `GTID_SUBTRACT`, et le réparer.

### 4.1 — Mettre en place la réplication (node1 = source, node2 = réplica)

Sur `node1`, créer un compte de réplication dédié :
```sql
CREATE USER IF NOT EXISTS 'repl'@'%' IDENTIFIED BY 'ReplPass2026!';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
```

Sur `node2`, pointer vers `node1` en auto-position GTID :
```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='<IP_PRIVEE_NODE1>',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='ReplPass2026!',
  SOURCE_AUTO_POSITION=1;
START REPLICA;
SHOW REPLICA STATUS\G
```
> **Question :** que valent `Replica_IO_Running` et `Replica_SQL_Running` ?
> Que doivent-ils afficher pour une réplication saine ?

### 4.2 — Vérifier la synchronisation

Sur `node1` :
```sql
INSERT INTO tp_gtid.compteur VALUES (2, 200);
```
Sur `node2`, quelques secondes après :
```sql
SELECT * FROM tp_gtid.compteur;
SELECT @@GLOBAL.gtid_executed;
```
> **Question :** comparez le `gtid_executed` de `node2` à celui de `node1`
> (exercice 1/2). En quoi `node2` a-t-il maintenant un GTID Set qui
> **contient** celui de `node1`, tout en ayant potentiellement des entrées
> en plus ?

### 4.3 — Casser la réplication (écriture directe sur le réplica)

Sur `node2`, **directement** (en contournant volontairement la réplication,
ce qu'une vraie architecture Group Replication empêcherait via le
`super_read_only` — voir Module 3) :
```sql
STOP REPLICA;
INSERT INTO tp_gtid.compteur VALUES (3, 999);
START REPLICA;
```
Puis sur `node1` :
```sql
INSERT INTO tp_gtid.compteur VALUES (3, 300);
```
Attendez quelques secondes, puis sur `node2` :
```sql
SHOW REPLICA STATUS\G
```
> **Question :** que montre `Last_SQL_Error` ? Pourquoi ce conflit
> survient-il précisément sur la ligne `id=3` ?

### 4.4 — Diagnostiquer avec GTID_SUBTRACT

Récupérez `@@GLOBAL.gtid_executed` sur `node1` ET sur `node2`, puis sur
`node1` :
```sql
SELECT GTID_SUBTRACT('<GTID_SET_NODE2>', '<GTID_SET_NODE1>');
```
> **Question :** le résultat correspond-il au GTID de la transaction locale
> que vous avez insérée directement sur `node2` à l'étape 4.3 ? C'est cette
> transaction "en trop", jamais vue par `node1`, qui bloque la réplication.

### 4.5 — Réparer

Deux approches possibles — testez la première :

**Option A — ignorer la transaction en conflit** (la donnée de `node2` sur
cette ligne prévaut, celle de `node1` est perdue pour ce réplica) :
```sql
-- Sur node2
STOP REPLICA;
SET GTID_NEXT = '<GTID_DE_LA_TRANSACTION_node1_QUI_ECHOUE>';
BEGIN; COMMIT;   -- "injecte" un GTID vide pour combler le trou côté application
SET GTID_NEXT = 'AUTOMATIC';
START REPLICA;
SHOW REPLICA STATUS\G
```
> **Question :** `Replica_SQL_Running` est-il repassé à `Yes` ? Que
> constatez-vous sur la valeur de `compteur` pour `id=3` côté `node2`
> maintenant ?

### 4.6 — Nettoyage avant de poursuivre la formation

Avant le Module 4 (création du cluster), remettez les nœuds dans un état
propre :
```sql
-- Sur node2
STOP REPLICA;
RESET REPLICA ALL;

-- Sur node1 ET node2
DROP DATABASE IF EXISTS tp_gtid;
```

## Exercice 5 (bonus) — Simuler un reset

Dans le contexte de l'ajout d'un nouveau noeud et de l'execution des requêtes suivantes :
```sql
RESET BINARY LOGS AND GTIDS;
SHOW VARIABLES LIKE 'gtid_executed';
```
> **Question :** que se passerait-il si ce nœud était actuellement membre
> d'un Group Replication au moment du reset ?

## Corrections

<details>
<summary>Voir les corrections</summary>

1. Le dernier numéro correspond au `transaction_id`, ici `2` (transaction 1 =
   `CREATE TABLE`, transaction 2 = `INSERT`, chacune commitée
   individuellement en mode autocommit).
2. Chaque serveur génère son `server_uuid` **indépendamment** à son premier
   démarrage (fichier `auto.cnf`) : deux installations distinctes ont donc
   nécessairement des UUID différents, même si le contenu des données est
   identique.
3. Le résultat est **le GTID Set complet de node1** : puisque node2 n'a
   jamais reçu aucune transaction de node1 (source_id différent),
   l'intersection est vide et la soustraction ne retire rien.
4.1. `Replica_IO_Running: Yes` et `Replica_SQL_Running: Yes` indiquent une
   réplication saine (le thread I/O reçoit le flux binlog, le thread SQL
   l'applique).
4.2. `node2` a désormais un GTID Set qui contient celui de `node1` (les
   transactions reçues portent le `source_id` de `node1`), et n'a
   normalement rien "en plus" à ce stade — les deux ensembles sont
   identiques puisque `node2` n'a fait qu'appliquer ce qu'il a reçu.
4.3. `Last_SQL_Error` indique un doublon de clé primaire (`Duplicate entry
   '3' for key 'PRIMARY'`) : `node2` a déjà une ligne `id=3` (celle insérée
   localement, en contournant la réplication), donc quand `node1` essaie de
   répliquer sa propre transaction `id=3`, l'application échoue.
4.4. Oui : le GTID "en trop" sur `node2` correspond exactement à la
   transaction insérée directement à l'étape 4.3 — c'est précisément ce que
   Group Replication automatise via la certification par writeset (Module
   3.7/3.9) pour éviter ce genre de situation dès l'écriture, plutôt que de
   la détecter après coup comme ici.
4.5. `Replica_SQL_Running` doit repasser à `Yes`. La valeur de `compteur`
   pour `id=3` sur `node2` reste `999` (la donnée insérée localement) : la
   transaction de `node1` a été "sautée" en injectant un GTID vide à sa
   place — c'est une réparation qui **accepte une perte de donnée
   contrôlée**, à documenter/valider avant de faire ce choix en production.
5. Un nœud actif de Group Replication ne doit **jamais** subir un reset GTID
   à chaud : le groupe le considérerait comme incohérent et
   l'expulserait (Module 3.10 — Gestion des conflits / limites
   transactionnelles).

</details>
