## Importer les data dans neo4j

La marche général est : 

1. Télécharger les export sql de wikipedia
2. Importer les export dans un base mysql
3. Exporter un csv pour neo4j
4. Importer les csv

### Télécharger les export sql

```bash
# en
wget https://dumps.wikimedia.org/enwiki/latest/enwiki-latest-linktarget.sql.gz
wget https://dumps.wikimedia.org/enwiki/latest/enwiki-latest-page.sql.gz
wget https://dumps.wikimedia.org/enwiki/latest/enwiki-latest-pagelink.sql.gz

# fr
wget https://dumps.wikimedia.org/frwiki/latest/frwiki-latest-linktarget.sql.gz
wget https://dumps.wikimedia.org/frwiki/latest/frwiki-latest-page.sql.gz
wget https://dumps.wikimedia.org/frwiki/latest/frwiki-latest-pagelink.sql.gz
```

### Importer les sql

#### split les fichier sql

On commence par split les export pour avoir un import threadé et donc beaucoup plus rapide : 

Pour chaque fichier : 

- Créer un dossier + mettre le fichier dedans
- Prendre le debut du fichier
```bash
# Adapter le nombre de ligne
head -39 enwiki-latest-linktarget.sql > init.sql
sed -i 1,39d enwiki-latest-linktarget.sql
```
- Prendre la fin du fichier
```bash
# Adapter le nombre de ligne
tail -12 enwiki-latest-linktarget.sql > end.sql
head -n -12 enwiki-latest-linktarget.sql > tmp.sql
mv tmp.sql enwiki-latest-linktarget.sql
```
- Split le fichier en x (selon le nombre de coeur de la machine)
```bash
trouver la taille en mega du fichier:
ls -h enwiki-latest-linktarget.sql
# diviser en x pour le nombre de thread et remplacer 392 par le nombre de mega divisé par x
split -l 392 enwiki-latest-linktarget.sql splited_
```

#### start un serveur mysql : 

```yaml
services:
  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 'root'
    command: --innodb-flush-log-at-trx-commit=0 --innodb-doublewrite=0 --innodb-buffer-pool-size=10G --innodb-log-file-size=10G
    volumes:
      - ./data:/var/lib/mysql
      - ./imports:/imports
      - ./export:/var/lib/mysql-files
    ports:
      - "3306:3306"
```

#### import sql

Répéter ces etapes pour chaque fichier
```bash
docker compose exec -it db bash
mysql -u root -proot
    create db en;
    create db fr;
ctrl + D

cd /import/{NOMDUFICHIER}
mysql -u root -proot en < init.sql
ls splited_a* | xargs -P12 -I{} bash -c "mysql -f -u root -proot en < {}"
mysql -u root -proot en < end.sql
```

### Export la data : 

Creer le script suivant dans export (ATTENTION VOUS DEVEZ MODIFIER LES VARIABLE EN DEBUT DE FICHIER)
```bash
#!/bin/bash

# Database credentials
DB_USER="root"
DB_PASS="root"
DB_NAME="en"
OUTPUT_DIR="/var/lib/mysql-files"
OUTPUT_FILE="${OUTPUT_DIR}/pagelinks.csv"
TMP_FILE_PREFIX="${OUTPUT_DIR}/pagelinks_part_"
THREADS=12


# Calculate chunk size
TOTAL_ROWS=63827087
CHUNK_SIZE=$(( (TOTAL_ROWS + THREADS - 1) / THREADS ))

# Create output directory if it doesn't exist
mkdir -p "$OUTPUT_DIR"

# Function to export a chunk
export_chunk() {
    local start_id=$1
    local end_id=$2
    local part_num=$3
    local tmp_file="${TMP_FILE_PREFIX}${part_num}.csv"

    mysql -u"$DB_USER" -p"$DB_PASS" "$DB_NAME" <<EOF
    SELECT p_from.page_id AS source_id, p_to.page_id AS target_id
    INTO OUTFILE '$tmp_file'
    FIELDS TERMINATED BY ',' ENCLOSED BY '"' LINES TERMINATED BY '\n'
    FROM pagelinks pl
    JOIN page p_from ON pl.pl_from = p_from.page_id
    JOIN linktarget lt ON pl.pl_target_id = lt.lt_id
    JOIN page p_to ON p_to.page_namespace = lt.lt_namespace AND p_to.page_title = lt.lt_title
    WHERE p_from.page_id BETWEEN $start_id AND $end_id;
EOF
    echo "Exported chunk $part_num: $start_id to $end_id"
}

# Launch threads
for ((i=0; i<THREADS; i++)); do
    start_id=$((i * CHUNK_SIZE + 1))
    end_id=$(( (i + 1) * CHUNK_SIZE ))
    # Ensure end_id does not exceed total rows
    if [ $end_id -gt $TOTAL_ROWS ]; then
        end_id=$TOTAL_ROWS
    fi
    export_chunk $start_id $end_id $i &
done

# Wait for all threads to finish
wait

# Concatenate all parts
cat "${TMP_FILE_PREFIX}"*.csv > "$OUTPUT_FILE"

# Clean up temporary files
rm -f "${TMP_FILE_PREFIX}"*.csv

echo "All chunks exported and merged into $OUTPUT_FILE"
```

Export les pages : 
```sql
SELECT page_id, CONVERT(page_title USING utf8mb4) INTO OUTFILE '/var/lib/mysql-files/page.csv' FIELDS TERMINATED BY ',' ENCLOSED BY '"' LINES TERMINATED BY '\n'from page limit 100;
```

Export les page links : Lancer les script avec les bonne variables


Ajouter des header sur les csv : 

```bash
[root@jonquille:/persistent/wikilinks/neo4j/imports]# head frpage.csv 
id:ID(Page-ID),title:STRING
222657,"!"
4433171,"!!"
351979,"!!!"
11632234,"!!!_(album)"
14399476,"!="
16174485,"!Action_Pact!"
16174504,"!Action_Pact!_(groupe)"
16286686,"!K7_Records"
11108328,"!Karas"

[root@jonquille:/persistent/wikilinks/neo4j/imports]# head enpage.csv 
id:ID(Page-ID),title:STRING
5878274,!
3632887,!!
600744,!!!
63743203,!!!!!!!
34443176,!!!Fuck_You!!!
11011780,!!!Fuck_You!!!_And_Then_Some
34443184,!!!Fuck_You!!!_and_Then_Some
59827823,!!!_(!!!_album)
59827819,!!!_(American_band)

[root@jonquille:/persistent/wikilinks/neo4j/imports]# head frpagelinks.csv 
:START_ID(Page-ID),:END_ID(Page-ID),:TYPE
3,9251052,LINKS_TO
3,349553,LINKS_TO
3,13419,LINKS_TO
3,318693,LINKS_TO
3,3566,LINKS_TO
3,1095,LINKS_TO
3,3789,LINKS_TO
3,4667,LINKS_TO
3,3495,LINKS_TO

[root@jonquille:/persistent/wikilinks/neo4j/imports]# head enpagelinks.csv 
:START_ID(Page-ID),:END_ID(Page-ID),:TYPE
10,13231,LINKS_TO
10,5611746,LINKS_TO
10,51841793,LINKS_TO
10,11442989,LINKS_TO
10,25520560,LINKS_TO
10,45397868,LINKS_TO
10,1192566,LINKS_TO
10,10782860,LINKS_TO
10,2822501,LINKS_TO
```

Importer les fichier dans la base neo4j (seul base accessible en community edition)

```
sudo docker exec -it neo4j-neo4j-1 bash
root@fbd63cbe6d3f:/var/lib/neo4j# apt update ; apt install -y sudo
...
root@fbd63cbe6d3f:/var/lib/neo4j# sudo -E -u neo4j bash -c '
export JAVA_TOOL_OPTIONS="--add-opens=java.base/java.nio=ALL-UNNAMED";
/var/lib/neo4j/bin/neo4j-admin database import full \
  --overwrite-destination=true \
  --nodes=import/enpage.csv \
  --relationships=import/enpagelinks.csv \
  --delimiter="," \
  --quote="\"" \
  neo4j
'

Picked up JAVA_TOOL_OPTIONS: --add-opens=java.base/java.nio=ALL-UNNAMED
Picked up JAVA_TOOL_OPTIONS: --add-opens=java.base/java.nio=ALL-UNNAMED
Starting to import, output will be saved to: /logs/neo4j-admin-import-2025-10-17.07.53.50.log
Neo4j version: 2025.09.0
Importing the contents of these files into /data/databases/neo4j:
Nodes:
  /var/lib/neo4j/import/enpage.csv

Relationships:
  null:
  /var/lib/neo4j/import/enpagelinks.csv


Available resources:
  Total machine memory: 15.48GiB
  Free machine memory: 6.004GiB
  Max heap memory : 10.00GiB
  Max worker threads: 12
  Configured max memory: 2.670GiB
  High parallel IO: true

WARN: file group with header file /var/lib/neo4j/import/enpage.csv specifies no node labels, which could be a mistake

Import starting 2025-10-17 07:53:59.979+0000
  Estimated number of nodes: 62.05 M
  Estimated number of node properties: 124.11 M
  Estimated number of relationships: 1.44 G
  Estimated number of relationship properties: 0.00 
  Estimated disk space usage: 51.28GiB
  Estimated required memory usage: 1.823GiB

(1/4) Node import 2025-10-17 07:53:59.988+0000
  Estimated number of nodes: 62.05 M
  Estimated disk space usage: 5.674GiB
  Estimated required memory usage: 1.823GiB
.......... .......... .......... .......... ..........   5% ∆11s 495ms [11s 495ms]
.......... .......... .......... .......... ..........  10% ∆4s 420ms [15s 916ms]
.......... .......... .......... .......... ..........  15% ∆1m 258ms [1m 16s 175ms]
.......... .......... .......... .......... ..........  20% ∆23s 425ms [1m 39s 600ms]
.......... .......... .......... .......... ..........  25% ∆6s 613ms [1m 46s 214ms]
.......... .......... .......... .......... ..........  30% ∆3s 405ms [1m 49s 620ms]
.......... .......... .......... .......... ..-.......  35% ∆14s 853ms [2m 4s 473ms]
.......... .......... .......... .......... ..........  40% ∆400ms [2m 4s 874ms]
.......... .......... .......... .......... ..........  45% ∆201ms [2m 5s 75ms]
.......... .......... .......... .......... ..........  50% ∆400ms [2m 5s 475ms]
.......... .......... .......... .......... ..........  55% ∆2s 203ms [2m 7s 679ms]
.......... .......... .......... .......... ..........  60% ∆1s 805ms [2m 9s 484ms]
.......... .......... .......... .......... ..........  65% ∆1s 803ms [2m 11s 288ms]
.......... .......... .......... .......... ..........  70% ∆1s 803ms [2m 13s 91ms]
.......... .......... .......... .......... ..........  75% ∆1s 402ms [2m 14s 494ms]
.......... .......... .......... .......... ..........  80% ∆200ms [2m 14s 694ms]
.......... .......... .......... .......... ..........  85% ∆200ms [2m 14s 895ms]
.......... .......... .......... .......... ..........  90% ∆272ms [2m 15s 168ms]
.......... .......... .......... .......... ..........  95% ∆0ms [2m 15s 168ms]
.......... .......... .......... .......... .......... 100% ∆0ms [2m 15s 168ms]
Node import COMPLETED in 2m 15s 172ms

(2/4) Relationship import 2025-10-17 07:56:15.160+0000
  Estimated number of relationships: 1.44 G
  Estimated disk space usage: 45.60GiB
  Estimated required memory usage: 1.710GiB
.......... .......... .......... .......... ..........   5% ∆26s 857ms [26s 857ms]
.......... .......... .......... .......... ..........  10% ∆30s 253ms [57s 110ms]
.......... .......... .......... .......... ..........  15% ∆51s 463ms [1m 48s 573ms]
.......... .......... .......... .......... ..........  20% ∆1m 9s 273ms [2m 57s 847ms]
.......... .......... .......... .......... ..........  25% ∆1m 47s 911ms [4m 45s 759ms]
.......... .......... .......... .......... ..........  30% ∆1m 14s 703ms [6m 462ms]
.......... .......... .......... .......... ..........  35% ∆46s 267ms [6m 46s 730ms]
.......... .......... .......... .......... ..........  40% ∆38s 452ms [7m 25s 183ms]
.......... .......... .......... .......... ..........  45% ∆28s 434ms [7m 53s 617ms]
.......... .......... .......... .......... ..........  50% ∆24s 632ms [8m 18s 250ms]
.......... .......... .......... .......... ..........  55% ∆23s 653ms [8m 41s 903ms]
.......... .......... .......... .......... ..........  60% ∆23s 640ms [9m 5s 543ms]
.......... .......... .......... .......... ..........  65% ∆22s 837ms [9m 28s 380ms]
.......... .......... .......... .......... ..........  70% ∆23s 39ms [9m 51s 420ms]
.......... .......... .......... .......... ..........  75% ∆26s 442ms [10m 17s 862ms]
.......... .......... .......... .......... ..........  80% ∆52s 251ms [11m 10s 113ms]
.......... .......... .......... .......... ..........  85% ∆30s 868ms [11m 40s 981ms]
.......... .......... .......... .......... ..........  90% ∆0ms [11m 40s 982ms]
.......... .......... .......... .......... ..........  95% ∆0ms [11m 40s 982ms]
.......... .......... .......... .......... .......... 100% ∆0ms [11m 40s 982ms]
Relationship import COMPLETED in 11m 40s 983ms

(3/4) Relationship linking 2025-10-17 08:07:56.143+0000
  Estimated required memory usage: 1.650GiB
.......... .......... .......... .......... ..........   5% ∆1m 25s 297ms [1m 25s 297ms]
.......... .......... .......... .......... ..........  10% ∆1m 1s 833ms [2m 27s 130ms]
.......... .......... .......... .......... ..........  15% ∆33s 420ms [3m 550ms]
.......... .......... .......... .......... .........-  20% ∆36s 870ms [3m 37s 421ms]
.......... .......... .......... .......... ..........  25% ∆1m 41s 653ms [5m 19s 74ms]
.......... .......... .......... .......... ..........  30% ∆2m 42s 675ms [8m 1s 750ms]
.......... .......... .......... .......... ..........  35% ∆1m 29ms [9m 1s 779ms]
.......... .......... .......... .......... ..........  40% ∆2m 31s 817ms [11m 33s 597ms]
.......... .......... .......... .......... ..........  45% ∆2m 15s 663ms [13m 49s 261ms]
.......... .......... .......... .......... ..........  50% ∆1m 43s 453ms [15m 32s 714ms]
.......... .......... .......... .......... ..........  55% ∆2m 3s 856ms [17m 36s 571ms]
.......... .......... .......... .......... .........-  60% ∆3m 2s 759ms [20m 39s 331ms]
.......... .......... .......... .......... ..........  65% ∆1m 42s 129ms [22m 21s 460ms]
.......... .......... .......... .......... ..........  70% ∆3m 57s 926ms [26m 19s 386ms]
.......... .......... .......... .......... ..........  75% ∆4m 22s 528ms [30m 41s 914ms]
.......... .......... .......... .......... ..........  80% ∆4m 43s 534ms [35m 25s 449ms]
.......... .......... .......... .......... ..........  85% ∆1m 11s 844ms [36m 37s 293ms]
.......... .......... .......... .......... ..........  90% ∆4m 23s 932ms [41m 1s 226ms]
.......... .......... .......... .......... ..........  95% ∆2m 3s 458ms [43m 4s 685ms]
.......... .......... .......... .......... .......... 100% ∆1m 28s 213ms [44m 32s 898ms]
Relationship linking COMPLETED in 44m 32s 903ms

(4/4) Post processing 2025-10-17 08:52:29.268+0000
  Estimated required memory usage: 1020MiB
...-....-. ...-...... .......... .......... ..........   5% ∆59s 119ms [59s 119ms]
.........- .......... .......... .......... ..........  10% ∆14s 807ms [1m 13s 927ms]
.......... .......... .......... .......... ..........  15% ∆14s 831ms [1m 28s 758ms]
.......... .......... .......... .......... ..........  20% ∆20s 497ms [1m 49s 256ms]
.......... .......... .......... .......... ..........  25% ∆24s 816ms [2m 14s 72ms]
.......... .......... .......... .......... ..........  30% ∆25s 614ms [2m 39s 687ms]
.......... .......... .......... .......... ..........  35% ∆16s 10ms [2m 55s 697ms]
.......... .......... .......... .......... ..........  40% ∆14s 608ms [3m 10s 306ms]
.......... .......... .......... .......... ..........  45% ∆17s 610ms [3m 27s 916ms]
.......... .......... .......... .......... ..........  50% ∆17s 11ms [3m 44s 927ms]
.......... .......... .......... .......... ..........  55% ∆17s 210ms [4m 2s 137ms]
.......... .......... .......... .......... ..........  60% ∆19s 611ms [4m 21s 748ms]
.......... .......... .......... .......... ..........  65% ∆22s 412ms [4m 44s 161ms]
.......... .......... .......... .......... ..........  70% ∆23s 813ms [5m 7s 974ms]
.......... .......... .......... .......... ..........  75% ∆22s 212ms [5m 30s 186ms]
.......... .......... .......... .......... ..........  80% ∆21s 811ms [5m 51s 998ms]
.......... .......... .......... .......... ..........  85% ∆24s 612ms [6m 16s 611ms]
.......... .......... .......... .......... ..........  90% ∆22s 211ms [6m 38s 823ms]
.......... .......... .......... .......... ..........  95% ∆20s 411ms [6m 59s 234ms]
.......... .......... .......... .......... .......... 100% ∆22s 694ms [7m 21s 928ms]
Post processing COMPLETED in 7m 22s 167ms


IMPORT DONE in 1h 5m 53s 10ms. 
Imported:
  63827087 nodes
  1171878848 relationships
  127654174 properties
Peak memory usage: 1.819GiB
root@fbd63cbe6d3f:/var/lib/neo4j# 
exit


