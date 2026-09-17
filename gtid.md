# TP — Manipulation des GTID

**Prérequis :** environnement du Module 0 provisionné, connexion SSH sur
`node1`, `node2`, `node3`.

### Exercice 1 — Observer la génération de GTID

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

### Exercice 2 — Comparer GTID_EXECUTED entre 2 serveurs non répliqués

Exécutez la même requête sur `node2` (qui n'est pas encore répliqué à ce
stade). Comparez les deux `server_uuid` dans les GTID obtenus.
> **Question :** pourquoi les deux GTID Sets utilisent-ils un `source_id`
> différent bien que la structure des données soit identique ?

### Exercice 3 — GTID_SUBTRACT en pratique

En utilisant les deux GTID Sets récupérés à l'exercice 2, exécutez sur
`node1` :
```sql
SELECT GTID_SUBTRACT(@@GLOBAL.gtid_executed, '<GTID_SET_DE_NODE2>');
```
> **Question :** que représente le résultat ? Est-ce cohérent avec le fait
> que les deux serveurs n'ont jamais été synchronisés ?

### Exercice 4 (bonus) — Simuler un reset

Sur un nœud de test **non utilisé pour le reste de la formation** (ou une 5e
instance jetable), testez :
```sql
RESET BINARY LOGS AND GTIDS;
SHOW VARIABLES LIKE 'gtid_executed';
```
> **Question :** que se passerait-il si ce nœud était actuellement membre
> d'un Group Replication au moment du reset ?

### Corrections

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
4. Un nœud actif de Group Replication ne doit **jamais** subir un reset GTID
   à chaud : le groupe le considérerait comme incohérent et
   l'expulserait (Module 3.10 — Gestion des conflits / limites
   transactionnelles).

</details>
