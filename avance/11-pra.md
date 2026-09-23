# TP 11 — Scénario PRA complet : perte du cluster primaire, bascule et restauration

**Module associé :** [Module 11 — Dépannage ClusterSet et PRA](../modules/module-11-depannage-clusterset-pra.md)

**Cet atelier est la synthèse de toute la formation.** Il combine
diagnostic GTID (Module 1), lecture des métriques internes (Modules 2-3,
5), AdminAPI (Module 4), Router (Module 7), clonage (Module 8) et
ClusterSet (Modules 9-10) dans un seul scénario de crise chronométré.

**Format recommandé :** en binôme, avec un rôle "incident commander" (qui
décide) et un rôle "exécutant" (qui tape les commandes) — représentatif
d'une astreinte réelle. Temps cible : 45-60 minutes.

## Contexte du scénario

`productionClusterSet` est en fonctionnement normal : `trainingCluster`
(site A) est PRIMARY, `siteB-cluster` (site B) est REPLICA. L'application
écrit en continu (relancez la boucle d'INSERT utilisée aux TP précédents
en arrière-plan, depuis `router1`, pour simuler une charge réelle pendant
tout le scénario).

## Étape 0 — Établir la situation de référence (avant incident)

Avant de déclencher quoi que ce soit, consignez :
```javascript
clusterSet.status({extended: 1});
```
```sql
-- Sur un membre de chaque cluster
SELECT @@GLOBAL.gtid_executed;
```
> Notez ces valeurs — elles serviront de point de comparaison pour évaluer
> la perte de données réelle en fin de scénario.

## Étape 1 — Incident : perte totale du site PRIMARY

Le formateur (ou l'apprenant en auto-formation) déclenche, sans prévenir le
binôme du timing exact :
```bash
# Sur les 3 membres de trainingCluster (site A)
sudo systemctl stop mysql
```

## Étape 2 — Diagnostic initial (rôle "incident commander")

1. Confirmer l'indisponibilité réelle (pas un faux positif de supervision) :
   ```javascript
   clusterSet.status({extended: 1});
   ```
2. Vérifier qu'aucune écriture ne passe plus côté application (le Router
   pointant vers le site A doit maintenant échouer).
3. **Décision à documenter :** bascule d'urgence justifiée ou attendre un
   peu (site potentiellement en train de revenir) ? Fixez un délai
   d'attente raisonnable avant de trancher (ex. 2 minutes dans ce TP).

## Étape 3 — Bascule d'urgence

```javascript
clusterSet.forcePrimaryCluster('siteB-cluster');
clusterSet.status();
```
Vérifiez immédiatement le fencing inversé :
```sql
-- Sur un membre de siteB-cluster
SELECT @@GLOBAL.super_read_only;   -- doit être 0 maintenant
```

## Étape 4 — Rediriger l'application

Si votre Router de test pointait spécifiquement vers le cluster A (selon
votre configuration `03-setup-router.sh`), rebootstrapez-le contre le
nouveau PRIMARY :
```bash
# Sur router1
sudo mysqlrouter --bootstrap clusteradmin@<IP_SITEB>:3306 \
  --directory /opt/mysqlrouter --conf-use-gr-notifications \
  --user=mysqlrouter --force
cd /opt/mysqlrouter && sudo ./stop.sh && sudo ./start.sh
```
Vérifiez que les écritures reprennent :
```bash
mysql -h127.0.0.1 -P6446 -uclusteradmin -p'ClusterAdmin2026!' \
  -e "SELECT @@hostname; INSERT INTO tp_gtid.compteur VALUES (11000, 1);"
```

## Étape 5 — Quantifier le RPO réel de l'incident

Comparez le `gtid_executed` de référence (Étape 0, site A) à celui du
nouveau PRIMARY (site B) au moment de la bascule :
```sql
SELECT GTID_SUBTRACT('<gtid_site_A_avant_incident>', '<gtid_site_B_au_moment_bascule>');
```
> **Question :** ce résultat représente les transactions du site A
> **jamais reçues** par le site B avant la coupure — donc définitivement
> perdues du point de vue applicatif. Combien de transactions cela
> représente-t-il dans votre scénario ? Ce chiffre est-il cohérent avec le
> volume généré par votre boucle de charge entre l'Étape 0 et l'Étape 1 ?

## Étape 6 — Retour nominal : réintégrer le site A

```bash
# Sur les 3 membres de trainingCluster (site A)
sudo systemctl start mysql
```
```javascript
clusterSet.status({extended: 1});
clusterSet.rejoinCluster('trainingCluster');
clusterSet.status({extended: 1});
```
> **Question :** le rejoin a-t-il nécessité un clone complet ou un rejeu
> incrémental ? Justifiez à partir du volume de transactions perdues
> calculé à l'Étape 5 (un site A qui avait beaucoup avancé seul avant la
> coupure a plus de transactions "orphelines" à écraser).

## Étape 7 — Décision finale : où reste le PRIMARY ?

> **Question de synthèse (débrief) :** maintenant que les deux sites sont
> à nouveau sains, faut-il rebasculer le PRIMARY vers le site A d'origine
> (`setPrimaryCluster()`, bascule planifiée) ou laisser le site B en
> PRIMARY durablement ? Listez au moins 2 arguments dans chaque sens
> (proximité applicative, coût d'une nouvelle bascule, risque de refaire
> l'exercice dans l'autre sens, etc.).

## Débrief de synthèse (tour de table, 15-20 min)

1. Quel a été le facteur limitant le plus dans ce scénario : la détection
   de l'incident, la décision de bascule, ou l'exécution technique ?
2. Si ce scénario s'était produit avec un ClusterSet à 3 sites (1 PRIMARY +
   2 REPLICA), qu'est-ce qui aurait changé dans la procédure de décision
   (Module 9.3) ?
3. Quelle métrique de supervision (Module 5), si elle avait été surveillée
   en continu avec alerte automatique, aurait permis de détecter
   l'incident plus tôt que par la panne applicative elle-même ?

## Corrections

<details>
<summary>Voir les corrections</summary>

Étape 5. Le nombre de transactions perdues correspond exactement à celles
   générées sur le site A entre le dernier point de synchronisation reçu
   par le site B et l'arrêt du site A — cohérent avec le débit de votre
   boucle de charge multiplié par le délai entre la dernière réplication
   effective et la coupure (pas tout le temps écoulé depuis l'Étape 0,
   seulement la partie non encore répliquée au moment T).

Étape 6. Plus le site A avait généré de transactions non répliquées avant
   la coupure (RPO élevé), plus `rejoinCluster()` tend à choisir un clone
   complet plutôt qu'un rejeu incrémental — reconstruire proprement depuis
   le nouveau PRIMARY plutôt que de tenter de réconcilier un historique
   trop divergent est le choix de sûreté par défaut.

Étape 7. Arguments typiques pour revenir au site A : proximité
   applicative/utilisateurs si l'app est majoritairement co-localisée avec
   le site A, cohérence avec la documentation d'architecture existante.
   Arguments pour rester sur le site B : éviter une nouvelle fenêtre de
   bascule (chaque bascule a un coût opérationnel et un risque résiduel),
   le site B a démontré sa capacité à assumer le rôle PRIMARY sans
   incident.

Débrief 3. Le lag du canal `clusterset_replication` (Module 9.2) ou le
   `MEMBER_STATE` des membres du site A (Module 5.2), surveillés avec
   alerte automatique, auraient signalé l'incident dès la coupure réseau
   ou l'arrêt des services — potentiellement plusieurs minutes avant que
   l'application ne remonte une erreur d'écriture côté utilisateur final.

</details>
