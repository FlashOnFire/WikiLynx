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

Creer le script suivant dans export (ATTENTION VOUS DEVRER MODIFIER LES VARIABLE EN DEBUT DE FICHIER)
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
