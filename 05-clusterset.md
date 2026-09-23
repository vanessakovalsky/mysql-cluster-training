# TP 5 — Mettre en place et tester le Router

## Exercice 1 — Bootstrap et test R/W vs R/O
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

## Exercice 2 — Observer la bascule automatique du Router lors d'un failover
1. Notez le PRIMARY actuel (`SELECT @@hostname` sur le port 6446).
2. Arrêtez MySQL sur ce nœud (`sudo systemctl stop mysql`).
3. Réinterrogez immédiatement le port 6446.
> **Question :** combien de temps le Router met-il à rediriger vers le
> nouveau PRIMARY ? Comparez avec/sans `--conf-use-gr-notifications`.

## Exercice 3 — API REST
```bash
curl -k https://localhost:8443/api/20190715/router/status | jq
```
> **Question :** quelle information cette API donne-t-elle qui n'est pas
> visible directement depuis `cluster.status()` (MySQL Shell) ?

## Exercice 4 — Read/Write Splitting sur une connexion unique

**Objectif :** observer concrètement la promesse du Read/Write Splitting
(Module 5.4) — une **seule** connexion applicative, sans que le
développeur ait à choisir lui-même le port 6446 ou 6447.

### 4.1 — Ajouter une route de splitting

Sur `router1`, éditez `/opt/mysqlrouter/mysqlrouter.conf` et ajoutez cette
section (à la suite des routes existantes) :
```ini
[routing:trainingCluster_splitting]
bind_address = 0.0.0.0
bind_port = 6450
destinations = metadata-cache://trainingCluster/?role=PRIMARY_AND_SECONDARY
routing_strategy = round-robin
connection_sharing = 1
access_mode = auto
```
Redémarrez le Router :
```bash
cd /opt/mysqlrouter && sudo ./stop.sh && sudo ./start.sh
```

### 4.2 — Observer le routage automatique dans UNE session

Toujours sur la **même connexion** (port 6450 uniquement), enchaînez ces
requêtes :
```bash
mysql -h127.0.0.1 -P6450 -uclusteradmin -p'ClusterAdmin2026!' <<'EOF'
SELECT @@hostname AS lecture_1;
UPDATE tp_gtid.compteur SET valeur = valeur + 1 WHERE id = 1;
SELECT @@hostname AS lecture_juste_apres_ecriture;
EOF
```
Puis, dans une **nouvelle** connexion (toujours sur 6450, mais après un
délai de quelques secondes) :
```bash
mysql -h127.0.0.1 -P6450 -uclusteradmin -p'ClusterAdmin2026!' \
  -e "SELECT @@hostname AS lecture_nouvelle_session;"
```

> **Question 1 :** `lecture_1` provient-elle toujours du PRIMARY, ou parfois
> d'un SECONDARY ?
> **Question 2 :** `lecture_juste_apres_ecriture` — sur quel nœud
> s'exécute-t-elle ? Pourquoi le Router ne bascule-t-il pas cette lecture
> vers un SECONDARY comme il l'a fait pour `lecture_1`, alors qu'elles sont
> écrites de façon identique en SQL ?
> **Question 3 :** `lecture_nouvelle_session` (nouvelle connexion) peut-elle
> à nouveau atterrir sur un SECONDARY ? Qu'est-ce que ça vous apprend sur la
> portée de la garantie de cohérence du Read/Write Splitting ?

## Corrections

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
4.1. `lecture_1` peut provenir d'un SECONDARY (le Router route les lectures
   vers `PRIMARY_AND_SECONDARY` par défaut quand rien ne l'en empêche
   encore dans la session).
4.2. `lecture_juste_apres_ecriture` reste sur le **PRIMARY** : le Router
   détecte qu'une écriture vient d'avoir lieu dans cette session et bascule
   temporairement toutes les lectures suivantes vers le PRIMARY, pour
   garantir la cohérence "read-your-own-writes" (éviter qu'une lecture
   juste après une écriture ne retombe sur un SECONDARY pas encore à jour,
   la réplication de Group Replication n'étant pas instantanée côté
   application des SECONDARY malgré la certification synchrone à l'écriture).
4.3. Oui, une **nouvelle** connexion peut repartir sur un SECONDARY : la
   garantie de cohérence du Read/Write Splitting est **limitée à la
   session/connexion en cours**, pas globale. Une nouvelle connexion n'a
   pas connaissance de l'écriture précédente et peut donc à nouveau être
   routée vers n'importe quel membre disponible en lecture.

</details>
