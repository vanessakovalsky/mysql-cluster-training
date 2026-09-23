# TP 7 — Simulation de bascule primaire, analyse du routage et diagnostic via logs Router

**Module associé :** [Module 7 — MySQL Router en production](../modules/module-07-mysql-router-production.md)

## Exercice 1 — Observer le décalage "connexion existante vs nouvelle connexion"

### 1.1 — Ouvrir une connexion PERSISTANTE avant l'incident
Dans un terminal, ouvrez une session interactive (ne la fermez pas) :
```bash
mysql -h127.0.0.1 -P6446 -uclusteradmin -p'ClusterAdmin2026!'
```
```sql
SELECT @@hostname;   -- notez le PRIMARY actuel
```
**Laissez cette session ouverte.**

### 1.2 — Provoquer le failover
Dans un autre terminal :
```bash
# Sur le PRIMARY actuel identifié en 1.1
sudo systemctl stop mysql
```

### 1.3 — Comparer connexion existante vs nouvelle connexion
**Sans fermer** la session de 1.1, tentez :
```sql
INSERT INTO tp_gtid.compteur VALUES (8000, 1);
```
> **Question :** que se passe-t-il ? Le message d'erreur mentionne-t-il
> `super_read_only` ou un problème de connexion ?

Dans un **nouveau** terminal :
```bash
mysql -h127.0.0.1 -P6446 -uclusteradmin -p'ClusterAdmin2026!' \
  -e "SELECT @@hostname; INSERT INTO tp_gtid.compteur VALUES (8001, 1);"
```
> **Question :** cette nouvelle connexion réussit-elle immédiatement ? En
> quoi ça confirme le comportement décrit au Module 7.4 (le Router route
> correctement les *nouvelles* connexions, mais ne migre pas les
> anciennes) ?

## Exercice 2 — Diagnostic via les logs Router

Pendant/juste après l'exercice 1 :
```bash
grep -iE "primary|metadata|notif" /opt/mysqlrouter/log/mysqlrouter.log | tail -30
```
> **Question :** à quel horodatage le Router a-t-il détecté le changement
> de PRIMARY ? Comparez avec l'horodatage de l'échec de la commande 1.3 —
> le Router avait-il déjà détecté la bascule avant que la connexion
> persistante n'échoue, ou après ?

## Exercice 3 — Chronométrer l'impact de use-gr-notifications

### 3.1 — Mesurer AVEC notifications (config actuelle)
```bash
# Remettre le nœud arrêté en route d'abord
sudo systemctl start mysql   # sur l'ancien PRIMARY, pour repartir propre
```
Notez le temps entre l'arrêt du PRIMARY et le moment où une **nouvelle**
connexion sur le port 6446 renvoie le nouveau `@@hostname` (script en
boucle avec `date` avant/après, ou chronomètre manuel).

### 3.2 — Désactiver temporairement les notifications et remesurer
```bash
# Sur router1 - relancer un bootstrap SANS conf-use-gr-notifications,
# ou commenter use_gr_notifications=1 dans mysqlrouter.conf puis redémarrer
sudo sed -i 's/use_gr_notifications=1/use_gr_notifications=0/' /opt/mysqlrouter/mysqlrouter.conf
cd /opt/mysqlrouter && sudo ./stop.sh && sudo ./start.sh
```
Répétez la mesure du failover.

> **Question :** quel est l'écart de temps observé ? Est-il cohérent avec
> la valeur du `ttl` configuré dans `mysqlrouter.conf` ?

Remettez `use_gr_notifications=1` en fin d'exercice.

## Corrections

<details>
<summary>Voir les corrections</summary>

1.3. L'`INSERT` sur la connexion persistante échoue avec une erreur liée
   au mode lecture seule (`super_read_only`) — le Router n'a pas coupé
   cette connexion, elle continue de pointer vers l'ancien PRIMARY devenu
   SECONDARY, qui refuse maintenant l'écriture. La **nouvelle** connexion,
   elle, réussit immédiatement : le Router l'a correctement dirigée vers
   le nouveau PRIMARY dès l'établissement.
2. Le Router détecte généralement le changement de PRIMARY **avant** ou
   en même temps que l'échec de la connexion persistante (la détection
   côté Router est rapide, en particulier avec les notifications GR) —
   mais cette détection ne rétroagit pas sur les connexions déjà ouvertes,
   ce qui est précisément le point du Module 7.4.
3. Sans notifications, l'écart de détection correspond approximativement
   au `ttl` configuré (temps d'attente avant le prochain rafraîchissement
   périodique de la metadata) — potentiellement plusieurs fois le `ttl` si
   la bascule survient juste après un rafraîchissement. Avec les
   notifications actives, la détection est quasi-immédiate (push), peu
   dépendante du `ttl`.

</details>
