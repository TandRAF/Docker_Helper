# Docker Helper in PostgresSQL
Cluster in MySQL is a to copy your main database throw other databases, so if one of the broke up u can use another, in our example we will have this folder structure ( Master - > Slave ) :
```
./data
./primary
	setup-replication.sh
docker-compose.yml
```
## Define the containers using docker-compose
first of all every cluster in a container, so in our `docker-compose.yml` we have to define details of all this components:
```yml
version: '3.8'
services:
  master:
    image: mysql:8.0
    container_name: mysql-master
    environment:
      MYSQL_ROOT_PASSWORD: root
    ports:
      - "3306:3306"
    volumes:
      - ./master/my.cnf:/etc/mysql/my.cnf
      - ./data:/var/lib/mysql-files
  slave1:
    image: mysql:8.0
    container_name: mysql-slave-1
    environment:
      MYSQL_ROOT_PASSWORD: root
    ports:
      - "3307:3306"
    volumes:
      - ./slave1/my.cnf:/etc/mysql/my.cnf
  slave2:
    image: mysql:8.0
    container_name: mysql-slave-2
    environment:
      MYSQL_ROOT_PASSWORD: root
    ports:
      - "3308:3306"
    volumes:
      - ./slave2/my.cnf:/etc/mysql/my.cnf
```
## Ensure the data replication
PostgreSQL needs a user with the REPLICATION privilege to allow the replicas to connect.</br>
**Primary**`setup-replication.sh`:
```shell
#!/bin/bash
set -e
psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" <<-EOSQL
    CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'password';
EOSQL
# Allow replication connections in the config
echo "host replication replicator all md5" >> "$PGDATA/pg_hba.conf"
```
Not forget about making it executable:
```bash
chmod +x setup-replication.sh
``` 
## Installing data
Go to data folder and install your data: 
```shell
wget https://datasets.imdbws.com/name.basics.tsv.gz
```
Also to unzip it:
```shell
gunzip name.basics.tsv.gz
```
To run the containers:
```bash
docker-compose up -d
```
## Create Table and Database
In Postgres, the Replica is a physical copy. This means once you create a table on the Master, it appears automatically on the Slaves. You do not need to run the CREATE commands on the slaves manually.
Access the Primary terminal:

```shell
docker exec -it postgres-primary psql -U user -d imdb
```
Execute the SQL:

```SQL
CREATE TABLE name_basics (
    nconst VARCHAR(15) PRIMARY KEY,
    primaryName VARCHAR(255),
    birthYear VARCHAR(4),
    deathYear VARCHAR(4),
    primaryProfession VARCHAR(255),
    knownForTitles VARCHAR(255)
);
```
## Installing Data
```sql
COPY name_basics 
FROM '/var/lib/postgresql/csv_data/name.basics.tsv' 
DELIMITER E'\t' CSV HEADER;
```
## Indexing (Full-Text Search)
PostgreSQL handles full-text indexing using tsvector and GIN indexes instead of MySQL's MATCH/AGAINST.
```SQL
CREATE INDEX idx_fts_name ON name_basics USING gin(to_tsvector('english', primaryName));
```
Usage (Search for 'alley')
```sql
SELECT * FROM name_basics 
WHERE to_tsvector('english', primaryName) @@ to_tsquery('english', 'alley');
```
## Indexing for Reverse Search
Postgres also supports generated columns for reverse string indexing.
```sql
-- Add a stored generated column
ALTER TABLE name_basics 
ADD COLUMN name_reversed TEXT GENERATED ALWAYS AS (reverse(primaryName)) STORED;

-- Create index on the reversed column
CREATE INDEX idx_reverse_name ON name_basics(name_reversed);

-- Search using the index
SELECT * FROM name_basics WHERE name_reversed LIKE 'ronno%';
```