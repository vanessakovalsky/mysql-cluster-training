# TP 5 — Diagnostic d'un nœud en erreur et analyse croisée métriques/logs

## Exercice — Isolement réseau d'un membre et diagnostic complet

### 1. Provoquer l'incident (isolement réseau, pas arrêt du service)

Sur `node3`, bloquer le port GCS (33061) vers les 2 autres nœuds, **sans
arrêter MySQL** — pour simuler une coupure réseau plutôt qu'une panne
logicielle, distinction importante en diagnostic réel :
```bash
sudo iptables -A INPUT -p tcp --dport 33061 -j DROP
sudo iptables -A OUTPUT -p tcp --dport 33061 -j DROP
```

### 2. Observer, sans encore corriger

Depuis `router1`, à intervalles de quelques secondes :
```sql
SELECT MEMBER_ID, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```
> **Question :** par quels états `node3` passe-t-il, dans quel ordre
> (`ONLINE` → ? → ?) ? Notez les horodatages approximatifs de chaque
> transition.

### 3. Lire les logs au moment de la transition

Sur `node1` (qui reste dans le groupe majoritaire) :
```bash
sudo grep -iE "view|expel|quorum" /var/log/mysql/error.log | tail -30
```
Sur `node3` lui-même (isolé) :
```bash
sudo grep -iE "view|expel|quorum|majority" /var/log/mysql/error.log | tail -30
```
> **Question :** les messages sur `node3` évoquent-ils une perte de
> **majorité** locale ? Pourquoi (indice : `node3` seul face à `node1` +
> `node2` ne peut plus, de son point de vue, constituer une majorité) ?

### 4. Vérifier l'impact sur la disponibilité du cluster

Depuis `router1`, pendant que `node3` est isolé :
```sql
-- Sur node1 ou node2 (via le Router, port 6446)
INSERT INTO tp_gtid.compteur VALUES (6000, 1);
```
> **Question :** cette écriture réussit-elle ? Cohérent avec le quorum
> (Module 2.5) — 2 membres sur 3 restent majoritaires.

### 5. Corriger et réintégrer

```bash
# Sur node3
sudo iptables -D INPUT -p tcp --dport 33061 -j DROP
sudo iptables -D OUTPUT -p tcp --dport 33061 -j DROP
```
Depuis MySQL Shell :
```javascript
var cluster = dba.getCluster();
cluster.status();
// Si node3 ne rejoint pas automatiquement :
cluster.rejoinInstance('clusteradmin:ClusterAdmin2026!@<IP_NODE3>:3306');
```
> **Question :** `node3` a-t-il besoin d'une recovery complète (clone) ou
> incrémentale pour rattraper le retard accumulé pendant l'isolement ?
> Justifiez à partir de la durée de l'incident et du volume de
> transactions générées pendant ce temps.

## Corrections

<details>
<summary>Voir les corrections</summary>

2. Typiquement `ONLINE` → `UNREACHABLE` (détecté par les autres membres,
   quelques secondes) → `ERROR` ou expulsion effective après
   `group_replication_member_expel_timeout` (5s par défaut en 8.0 récent).
   `node3` lui-même peut se voir passer en état d'erreur local une fois
   qu'il détecte sa propre perte de majorité.
3. Oui, côté `node3`, les logs évoquent typiquement l'impossibilité de
   maintenir une vue majoritaire du groupe — `node3` ne voit plus
   `node1`/`node2`, et seul face à lui-même il représente 1 voix sur 3,
   donc pas de majorité. Côté `node1`, les logs évoquent au contraire un
   changement de vue réussi excluant `node3`, la majorité restant intacte
   à 2/3.
4. Oui, l'écriture réussit normalement : le cluster reste disponible en
   écriture avec 2 membres sur 3 (majorité conservée).
5. Pour un isolement court (quelques minutes, comme dans ce TP), une
   recovery **incrémentale** suffit généralement — le volume de
   transactions manquées reste faible et encore disponible dans les
   binlogs. Un isolement de plusieurs heures avec forte activité
   d'écriture basculerait plus probablement vers un clone automatique
   (Module 1.3, purge des binlogs dépassée).

</details>
