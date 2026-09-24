# TP 6 — Perte de quorum, redémarrage complet et remise en cohérence

## Exercice 1 — Perte de quorum (2 membres sur 3 tombent)

### 1.1 — Provoquer l'incident
```bash
# Sur node2 ET node3 simultanément
sudo systemctl stop mysql
```

### 1.2 — Constater l'indisponibilité en écriture
Depuis `router1` (port 6446) :
```sql
INSERT INTO tp_gtid.compteur VALUES (7000, 1);
```
> **Question :** quel message d'erreur obtenez-vous sur `node1` ? Est-ce le
> même type d'erreur qu'un simple problème réseau applicatif, ou un message
> spécifique à Group Replication ?

### 1.3 — Confirmer le diagnostic avant d'agir
```sql
-- Sur node1
SELECT * FROM performance_schema.replication_group_members;
```
> **Question :** combien de membres sont vus comme `ONLINE` ? Confirmez
> qu'on est bien dans un cas de perte de majorité (1 membre sur 3), pas
> une simple lenteur.

### 1.4 — Forcer le quorum sur le survivant
```javascript
shell.connect('clusteradmin:ClusterAdmin2026!@<IP_NODE1>:3306');
var cluster = dba.getCluster();
cluster.forceQuorumUsingPartitionOf('clusteradmin:ClusterAdmin2026!@<IP_NODE1>:3306');
cluster.status();
```
> **Question :** l'écriture testée en 1.2 fonctionne-t-elle maintenant ?
> Combien de membres `cluster.status()` affiche-t-il comme faisant partie
> du cluster à ce stade (attention, pas le même nombre qu'avant l'incident) ?

### 1.5 — Réintégrer les membres revenus
```bash
# Sur node2 et node3
sudo systemctl start mysql
```
```javascript
cluster.status();  // les 2 membres réapparaissent-ils seuls, ou faut-il agir ?
cluster.rejoinInstance('clusteradmin:ClusterAdmin2026!@<IP_NODE2>:3306');
cluster.rejoinInstance('clusteradmin:ClusterAdmin2026!@<IP_NODE3>:3306');
cluster.status();
```
> **Question :** pourquoi `forceQuorumUsingPartitionOf` nécessite-t-il une
> réintégration explicite, alors qu'un simple redémarrage réseau (TP5)
> permettait un retour automatique ?

## Exercice 2 — Remise en cohérence complète

Après l'exercice 1, vérifiez qu'aucune transaction errante ne s'est glissée
pendant l'incident (Module 1.4-1.6 — méthode de diagnostic) :
```sql
-- Sur chaque membre
SELECT @@GLOBAL.gtid_executed;
```
Comparez deux à deux avec `GTID_SUBTRACT`.
> **Question :** dans ce scénario (aucune écriture directe sur `node2`/
> `node3` pendant leur arrêt), attendez-vous une divergence ? Pourquoi
> est-ce différent d'un scénario où quelqu'un aurait écrit directement sur
> un membre isolé (comme au TP5) ?

## Corrections

<details>
<summary>Voir les corrections</summary>

1.2. Le message est spécifique à Group Replication (typiquement une
   erreur indiquant que le serveur n'est pas en mesure d'agir comme membre
   actif du groupe / lecture seule forcée par perte de quorum) — pas une
   simple erreur de connectivité réseau générique.
1.3. 1 seul membre `ONLINE` (`node1`) sur les 3 attendus — confirmation
   claire d'une perte de majorité (1/3, pas de quorum).
1.4. L'écriture fonctionne après `forceQuorumUsingPartitionOf`.
   `cluster.status()` affiche alors le cluster comme fonctionnant avec
   uniquement le(s) membre(s) listé(s) dans la commande — les autres sont
   marqués comme manquants/à réintégrer, pas automatiquement comptés.
1.5. `forceQuorumUsingPartitionOf` acte une **décision explicite** de
   redéfinir la majorité autour des survivants désignés — les membres
   exclus de cette décision ne peuvent pas rejoindre automatiquement une
   vue dont ils ne faisaient plus partie "officiellement" ; contrairement à
   une coupure réseau simple (TP5) où le groupe n'a jamais été
   reconfiguré de force, juste temporairement réduit par expulsion
   naturelle.
2. Aucune divergence attendue ici : `node2`/`node3` étaient simplement
   arrêtés, sans écriture locale pendant leur indisponibilité — au retour,
   ils n'ont "que" du retard à rattraper (recovery), pas de transaction
   errante à réconcilier. C'est fondamentalement différent d'un cas où un
   membre isolé continuerait, par erreur de configuration, à accepter des
   écritures locales pendant la coupure (super_read_only mal positionné).

</details>
