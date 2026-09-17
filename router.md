# TP — Mettre en place et tester le Router

### Exercice 1 — Bootstrap et test R/W vs R/O
```bash
# Sur router1
./aws/03-setup-router.sh 10.42.0.11
cd /opt/mysqlrouter && sudo ./start.sh
```
```bash
mysql -h127.0.0.1 -P6446 -uclusteradmin -p'ClusterAdmin2026!' \
  -e "SELECT @@hostname, @@read_only;"
mysql -h127.0.0.1 -P6447 -uclusteradmin -p'ClusterAdmin2026!' \
  -e "SELECT @@hostname, @@read_only;"
```
> **Question :** le port 6447 retourne-t-il toujours le même `@@hostname`
> quand vous répétez la commande plusieurs fois ? Pourquoi ?

### Exercice 2 — Observer la bascule automatique du Router lors d'un failover
1. Notez le PRIMARY actuel (`SELECT @@hostname` sur le port 6446).
2. Arrêtez MySQL sur ce nœud (`sudo systemctl stop mysql`).
3. Réinterrogez immédiatement le port 6446.
> **Question :** combien de temps le Router met-il à rediriger vers le
> nouveau PRIMARY ? Comparez avec/sans `--conf-use-gr-notifications`.

### Exercice 3 — API REST
```bash
curl -k https://localhost:8443/api/20190715/router/status | jq
```
> **Question :** quelle information cette API donne-t-elle qui n'est pas
> visible directement depuis `cluster.status()` (MySQL Shell) ?

### Corrections

<details>
<summary>Voir les corrections</summary>

1. Non : en `round-robin-with-fallback`, le Router répartit les lectures
   sur `node2` et `node3` tour à tour ; le `@@hostname` doit donc alterner
   à chaque connexion (ou selon la stratégie, à chaque requête si
   Read/Write Splitting activé).
2. Avec `--conf-use-gr-notifications`, la bascule est détectée en
   généralement moins d'une seconde (notification push). Sans cette option,
   le Router attend le prochain cycle de rafraîchissement `ttl` (configurable,
   souvent 0.5 à quelques secondes par défaut) avant de relire les
   métadonnées.
3. L'API REST donne une vue **côté Router** (connexions actives par route,
   santé perçue de chaque destination du point de vue du Router), alors que
   `cluster.status()` donne une vue **côté serveurs** (état interne de
   Group Replication). Les deux sont complémentaires en exploitation.

</details>
