Angelehnt an:
https://www.unifox.at/2021/10/19/zammad-datenbankwechsel-mysql-zu-postgres/

Migration von mySQL zu Postgres
Zammad Docker Stack klonen
git clone https://github.com/zammad/zammad-docker-compose.git

Dann .env anpassen:
RESTART=unless-stopped
VERSION=-4.1.0 # passend wählen!

#####

Dann docker-compose.override.yaml anpassen:
---
version: '3'

services:

  zammad-nginx:
    ports:
      - "8081:8080" # freien port wählen

  # Anpassungen hier nicht notwendig, s. migration-stack...
  #zammad-postgresql:
  #  volumes:
  #    - zammad-data:/opt/zammad  # NEU
  #    - zammad-backup:/var/tmp/zammad:ro # NEU      

  adminer:
    image: adminer
    restart: unless-stopped
    ports:
      - 8083:8080

####

Einmal starten: docker-compose up -d
first-steps durchlaufen.
Runterfahren: docker-compose down

####

Ordner "postgres-with-pgloader" erzeugen, darin ein Dockerfile
FROM postgres:9.6
RUN apt-get update
RUN apt-get install -y pgloader

#######

Eine Datei migrationstack.yml erzeugen (lokale Volumepfade auf Backup der zammad-Datenbank zeigen lassen):
---
version: '3'

services:
  db-mysql:
    image: mariadb:10.3
    container_name: mariadb-tmpZammad
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: notSecureChangeMe
      MYSQL_DATABASE: zammad
      MYSQL_USER: zammad
      MYSQL_PASSWORD: zammad
    volumes:
      - /srv/dev-disk-by-uuid-293b50c8-7937-4e00-906c-3175d8a81f3d/backups_KANNWEG/db-mysql_data:/var/lib/mysql
      - /srv/dev-disk-by-uuid-293b50c8-7937-4e00-906c-3175d8a81f3d/backups_KANNWEG:/tmp/backup

  db-postgres:
    build: postgres-with-pgloader
    container_name: postgres-tmpZammad
    restart: unless-stopped
    volumes:
      - postgresql-data:/var/lib/postgresql/data
      - /srv/dev-disk-by-uuid-293b50c8-7937-4e00-906c-3175d8a81f3d/backups_KANNWEG:/tmp/backup
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASS}
      
  adminer:
    image: adminer
    restart: unless-stopped
    ports:
      - 8083:8080

volumes:
  postgresql-data:
    driver: local

####

Hochfahren:
sudo docker-compose -f migrationstack.yml up -d --build

Mysql-Backup importieren
docker exec -it mariadb-tmpZammad /bin/bash
cd /tmp/backup
zcat latest_zammad_db.mysql.gz | mysql -uzammad -pzammad zammad

In Postgres übernehmen:
docker exec -it mariadb-tmpZammad /bin/bash
Datei dumpZammad erzeugen:
load database
   from mysql://zammad:zammad@mariadb-tmpZammad/zammad
   into pgsql://zammad@127.0.0.1/zammad_production
   alter schema 'zammad_production' rename to 'public'
   with batch concurrency = 1;

pgloader --dry-run dumpZammad
pgloader dumpZammad

Runterfahren:
sudo docker-compose -f migrationstack.yml down

Zammad neustarten
Neustarten: docker-compose up -d
sudo docker-compose exec -it zammad-railsserver /bin/bash
rails r "Setting.set('es_url', 'http://zammad-elasticsearch:9200')"
rails r Rails.cache.clear
rake searchindex:rebuild

