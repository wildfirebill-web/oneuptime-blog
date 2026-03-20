# How to Manage Database Backups with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Database Backup, MySQL, PostgreSQL, MongoDB, Automation

Description: Learn how to schedule and manage database backups for MySQL, PostgreSQL, and MongoDB containers running in Portainer stacks.

---

Database backups in Portainer-managed containers are handled via sidecar containers or cron jobs that run `mysqldump`, `pg_dump`, or `mongodump`. This guide provides ready-to-use backup patterns for the most common databases.

## MySQL Backup Sidecar

Add a `db-backup` service to your stack that dumps the database on a schedule:

```yaml
version: "3.8"

services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: appdb
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - app_net

  db-backup:
    image: mysql:8.0
    environment:
      MYSQL_HOST: mysql
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: appdb
    volumes:
      - ./backups:/backups
    entrypoint: >
      sh -c "while true; do
        echo 'Starting backup...';
        mysqldump -h$$MYSQL_HOST -uroot -p$$MYSQL_ROOT_PASSWORD $$MYSQL_DATABASE \
          | gzip > /backups/backup-$$(date +%Y%m%d-%H%M%S).sql.gz;
        echo 'Backup complete';
        find /backups -name '*.sql.gz' -mtime +7 -delete;
        sleep 86400;
      done"
    networks:
      - app_net

volumes:
  mysql_data:

networks:
  app_net:
```

## PostgreSQL Backup

Use `pg_dump` with a similar sidecar pattern:

```yaml
  pg-backup:
    image: postgres:15
    environment:
      PGHOST: postgres
      PGUSER: postgres
      PGPASSWORD: postgrespassword
      PGDATABASE: appdb
    volumes:
      - ./backups:/backups
    entrypoint: >
      sh -c "while true; do
        pg_dump -Fc > /backups/backup-$$(date +%Y%m%d-%H%M%S).dump;
        find /backups -name '*.dump' -mtime +7 -delete;
        sleep 86400;
      done"
    networks:
      - app_net
```

Restore a PostgreSQL backup with:

```bash
pg_restore -h localhost -U postgres -d appdb /backups/backup-20260320.dump
```

## MongoDB Backup

Use `mongodump` to create compressed BSON archives:

```yaml
  mongo-backup:
    image: mongo:7.0
    environment:
      MONGO_URI: "mongodb://admin:password@mongo:27017/appdb?authSource=admin"
    volumes:
      - ./backups:/backups
    entrypoint: >
      sh -c "while true; do
        mongodump --uri=$$MONGO_URI --archive=/backups/backup-$$(date +%Y%m%d-%H%M%S).gz --gzip;
        find /backups -name '*.gz' -mtime +7 -delete;
        sleep 86400;
      done"
    networks:
      - app_net
```

Restore a MongoDB backup with:

```bash
mongorestore --uri="mongodb://admin:password@localhost:27017" \
  --archive=/backups/backup-20260320.gz --gzip
```

## Uploading Backups to S3

Extend any backup sidecar to push files to S3 after creation:

```bash
# Add after the dump command:

aws s3 cp /backups/backup-$(date +%Y%m%d-%H%M%S).sql.gz \
  s3://my-db-backups/mysql/ --storage-class STANDARD_IA

# Or with MinIO:
mc cp /backups/backup-*.sql.gz minio/db-backups/mysql/
```

## On-Demand Backup via Docker CLI

Trigger an immediate backup without waiting for the scheduled interval:

```bash
# MySQL on-demand dump
docker exec $(docker ps -qf name=mysql) \
  mysqldump -uroot -prootpassword appdb | \
  gzip > /tmp/manual-backup-$(date +%Y%m%d).sql.gz

# PostgreSQL on-demand dump
docker exec $(docker ps -qf name=postgres) \
  pg_dump -U postgres -Fc appdb > /tmp/manual-backup.dump
```

## Monitoring Backup Jobs

Use OneUptime or Healthchecks.io to alert if a backup hasn't run on time. Add a ping at the end of each successful backup cycle:

```bash
curl -fsS --retry 3 https://healthchecks.io/ping/YOUR-UUID
```

This creates a dead-man's switch: if the backup container crashes or the schedule is disrupted, you receive an alert before data is lost.
