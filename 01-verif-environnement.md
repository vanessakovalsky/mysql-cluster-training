# TP 1 — Vérification de l'environnement

**Module associé :** [Module 0 — Environnement AWS](../modules/module-00-environnement-aws.md)

**Objectif :** s'assurer que chaque apprenant peut se connecter à ses 4
machines et que la configuration de base est bien en place, avant d'attaquer
le Module 1.

1. Récupérer dans `infra.env` les IP publiques des 4 machines qui vous sont
   attribuées.
2. Se connecter en SSH à `node1` :
   ```bash
   ssh -i mysql-innodb-cluster-training-key.pem ubuntu@<IP_PUBLIQUE_NODE1>
   ```
3. Vérifier que MySQL tourne et que GTID est actif :
   ```bash
   sudo mysql -uroot -p'FormationMySQL2026!' -e "SHOW VARIABLES LIKE 'gtid_mode'; SHOW VARIABLES LIKE 'server_id';"
   ```
4. Répéter sur `node2` et `node3` et vérifier que chaque `server_id` est
   **différent**.
5. Depuis `router1`, tester la connectivité réseau vers les 3 nœuds sur le
   port 3306 :
   ```bash
   for ip in <IP_PRIVEE_NODE1> <IP_PRIVEE_NODE2> <IP_PRIVEE_NODE3>; do
     mysql -h "$ip" -uclusteradmin -p'ClusterAdmin2026!' -e "SELECT @@hostname;"
   done
   ```

**Critère de réussite :** les 3 commandes `SELECT @@hostname` répondent sans
erreur depuis `router1` → le réseau et les comptes sont opérationnels, la
formation peut démarrer.
