# TP 3 — Simulation de charge et analyse du throttling

## Exercice 1 — Mesure de référence
* Lancer le meme watch que le TP2
* Puis insérer des données
```bash
time (for i in $(seq 1 3000); do
  mysql -uclusteradmin -p'ClusterAdmin2026!' \
    -e "INSERT INTO tp_gtid.compteur VALUES ($((2000+i)), $i);" 2>/dev/null
done)
```
Notez le temps `real`. Observez en parallèle :
```sql
SELECT MEMBER_ID, COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE
FROM performance_schema.replication_group_member_stats;
```

## Exercice 2 — Simuler un membre lent artificiellement (I/O dégradée)

Sur `node3` uniquement, ralentissez artificiellement les écritures disque
(sans y toucher directement, en simulant une charge I/O concurrente) :
```bash
# Génère une charge I/O de fond pendant le test suivant
sudo apt-get install -y fio
fio --name=stress --rw=randwrite --bs=4k --size=500M --numjobs=2 \
  --runtime=60 --time_based --filename=/tmp/fio-stress &
```
Relancez immédiatement la même charge d'écriture qu'à l'exercice 1 (nouvelle
plage d'ID pour éviter les doublons).

> **Question :** le temps `real` global augmente-t-il, même si la charge
> est appliquée à `node3` (un SECONDARY) et non au PRIMARY qui reçoit les
> écritures ? Expliquez avec le mécanisme de flow control (3.2).

## Exercice 3 — Isoler la cause : certifieur ou applier ?

Comparez, pendant la charge :
```sql
SELECT COUNT_TRANSACTIONS_ROWS_VALIDATING,          -- certifieur
       COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE   -- applier
FROM performance_schema.replication_group_member_stats
WHERE MEMBER_ID = @@server_uuid;
```
> **Question :** laquelle des deux files croît réellement sur `node3`
> pendant l'exercice 2 ? Qu'est-ce que ça élimine comme cause possible
> (le certifieur, qui dépend surtout du CPU, ou l'applier, qui dépend
> surtout de l'I/O disque) ?

## Exercice 4 — Absorber un pic ponctuel connu

Arrêtez le `fio` de l'exercice 2 (`kill %1` ou `pkill fio`). Puis, avant de
relancer un nouveau pic de charge, augmentez temporairement le seuil :
```sql
SET GLOBAL group_replication_flow_control_applier_threshold = 100000;
```
Relancez la charge (exercice 1, nouvelle plage d'ID). Comparez le temps
`real` à la mesure de référence.

> **Question :** le débit global remonte-t-il proche de la mesure de
> référence, malgré `node3` toujours en I/O dégradée si vous relancez
> `fio` en parallèle ? Quel est le vrai compromis pris en augmentant ce
> seuil (indice : que devient l'écart de données entre `node3` et les
> autres membres pendant ce laps de temps ?).

Remettez le seuil à sa valeur par défaut (25000) en fin d'exercice.

## Corrections

<details>
<summary>Voir les corrections</summary>

2. Oui, le temps `real` global augmente : le flow control ralentit
   **tous** les membres (y compris le PRIMARY qui reçoit les écritures) dès
   que la file d'un membre quelconque dépasse le seuil — c'est le mécanisme
   même de la back pressure (3.2). Un ralentissement localisé sur un
   SECONDARY devient donc un ralentissement perçu partout.
3. C'est `COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE` qui croît
   significativement, pas `COUNT_TRANSACTIONS_ROWS_VALIDATING` — cohérent
   avec une charge I/O disque qui ralentit l'application (écriture sur
   disque), pas la certification (calcul CPU sur les writesets). Ça oriente
   le diagnostic vers l'I/O du membre concerné plutôt que vers son CPU.
4. Le débit remonte proche de la référence même avec `node3` en I/O
   dégradée : en desserrant le seuil, on autorise `node3` à accumuler un
   retard plus important **sans ralentir les autres**. Le compromis est
   que l'écart entre `node3` et le reste du groupe grandit pendant ce
   temps — acceptable pour un pic ponctuel connu et borné, dangereux si
   maintenu en permanence (retard non maîtrisé, RTO de recovery plus long
   en cas de panne du PRIMARY pendant cette fenêtre).

</details>
