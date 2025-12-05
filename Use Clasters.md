# Docker Helper
Cluster in MySQL is a to copy your main database throw other databases, so if one of the broke up u can use another, in our example we will have this folder structure ( Master - > Slave ) :
```
./data
./master
	my.cnf
./slave
	my.cnf
./slave
	my.cnf
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
Each MySQL instance will use a separate `my.cnf` file to specify its unique configuration settings for replication. Below are the configurations for each instance:
**Master `my.cnf`**:
```cnf
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-do-db=testdb
```
**Slave 1 `my.cnf`**:
```cnf
[mysqld]
server-id=2
relay-log=relay-log-bin
read-only=1
```
**Slave 2 `my.cnf`**:
```cnf
[mysqld]
server-id=3
relay-log=relay-log-bin
read-only=1
```
to run containers:
```yml
docker-compose up -d
```
in the master, for starting replication input this code:
```sql 
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'password'; GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%'; FLUSH PRIVILEGES; SHOW MASTER STATUS;
```
## Create TABLE and DATABASES for both, master and slave
create the table in the master folder, and also slaves, for open mysql terminal where instruction have to be writen use this command:
for the master
```shell
docker exec -it mysql-master mysql -u root -p 
```
for slave (both of them)
```shell
docker exec -it mysql-slave-1 mysql -u root -p 
```
For each of them insert this commands
```sql
CREATE DATABASE imdb;
USE imdb;

CREATE TABLE name_basics (
    nconst VARCHAR(15) PRIMARY KEY,
    primaryName VARCHAR(255),
    birthYear VARCHAR(4),
    deathYear VARCHAR(4),
    primaryProfession VARCHAR(255),
    knownForTitles VARCHAR(255)
);
```
## Connect Master with Slaves 
first of all find out what is the Master data, for that exec master sql:
```shell
docker exec -it mysql-master mysql -u root -p
```
the show the master data:
```sql
SHOW MASTER STATUS;
```
u have to find the file name and position in the table, and save it for the slave sql:
```sql
CHANGE MASTER TO 
	MASTER_HOST='mysql-master', 
	MASTER_USER='repl', 
	MASTER_PASSWORD='password', 
	MASTER_LOG_FILE='mysql-bin.000003', -- Valoarea ta de la File
	MASTER_LOG_POS=890; -- Valoarea ta de la Position 
START SLAVE;
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
import data from the data folder in the master one
```sql
LOAD DATA INFILE '/var/lib/mysql-files/name.basics.tsv'
INTO TABLE name_basics
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
IGNORE 1 LINES;
```
## Indexing
```sql
ALTER TABLE name_basics ADD FULLTEXT INDEX ft_name (primaryName);
```
and for using 
```sql
SELECT * FROM name_basics WHERE MATCH(primaryName) AGAINST('alley');
```
## Indexing for reverse
```sql
ALTER TABLE name_basics ADD COLUMN name_reversed VARCHAR(255) GENERATED ALWAYS AS (REVERSE(primaryName)) STORED;

CREATE INDEX idx_reverse_name ON name_basics(name_reversed);

SELECT * FROM name_basics WHERE name_reversed LIKE 'ronno%';
```
