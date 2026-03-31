# How to Upgrade MySQL in Docker Without Data Loss

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Docker, Upgrade, Migration, Volume

Description: Safely upgrade a MySQL Docker container to a newer version by backing up data, pulling the new image, and letting MySQL auto-upgrade the data directory on first start.

---

## Understanding MySQL Docker Upgrades

Upgrading MySQL in Docker involves replacing the container image with a newer version while keeping the data volume intact. MySQL 8.0 and later includes an automatic upgrade mechanism: on the first start with a new binary against an older data directory, MySQL runs the upgrade process automatically. For major version upgrades (e.g., 5.7 to 8.0), additional preparation is required.

## Step 1 - Back Up Your Data

Never upgrade without a backup. Dump all databases before proceeding:

```bash
docker exec mysql mysqldump \
  -u root -proot_secret \
  --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  > /backup/mysql_pre_upgrade_$(date +%Y%m%d).sql
```

Also back up the volume as a file archive:

```bash
docker run --rm \
  -v mysql_data:/var/lib/mysql:ro \
  -v /backup:/backup \
  ubuntu \
  tar czf /backup/mysql_volume_$(date +%Y%m%d).tar.gz -C /var/lib/mysql .
```

## Step 2 - Check the Upgrade Path

For minor version upgrades (e.g., 8.0.35 to 8.0.40), in-place upgrade is safe. For major version upgrades (5.7 to 8.0), first check compatibility:

```bash
# On the running 5.7 container
docker exec mysql mysqlcheck \
  -u root -proot_secret \
  --all-databases \
  --check-upgrade
```

Resolve any reported issues before upgrading.

## Step 3 - Stop the Current Container (Not Remove)

```bash
docker compose down
# or
docker stop mysql
```

Do not use `docker compose down -v` - this would delete the volume.

## Step 4 - Update the Image Version

In `docker-compose.yml`, update the image tag:

```yaml
services:
  mysql:
    image: mysql:8.0.40   # was mysql:8.0.35
```

Or pull the new image directly:

```bash
docker pull mysql:8.0.40
```

## Step 5 - Start the New Container

```bash
docker compose up -d
# or
docker run -d \
  --name mysql \
  -v mysql_data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=root_secret \
  mysql:8.0.40
```

## Step 6 - Monitor the Upgrade Process

Watch the logs to confirm the upgrade completes successfully:

```bash
docker logs -f mysql 2>&1 | grep -E "Upgrade|upgrade|ERROR|ready for connections"
```

MySQL 8.0 logs a message like `Upgrading data directory...` followed by `ready for connections` when done.

## Step 7 - Run mysql_upgrade (MySQL 5.7 Only)

For MySQL 5.7 minor upgrades, run `mysql_upgrade` manually after starting:

```bash
docker exec -it mysql mysql_upgrade -u root -proot_secret
```

MySQL 8.0+ handles this automatically via the `--upgrade=AUTO` default.

## Step 8 - Verify the Upgrade

```bash
docker exec mysql mysql -u root -proot_secret \
  -e "SELECT VERSION(); SHOW DATABASES;"
```

## Rollback Plan

If the upgrade fails, restore from the volume backup:

```bash
docker compose down
docker volume rm mysql_data
docker volume create mysql_data
docker run --rm \
  -v mysql_data:/var/lib/mysql \
  -v /backup:/backup \
  ubuntu \
  tar xzf /backup/mysql_volume_<date>.tar.gz -C /var/lib/mysql
docker compose up -d   # with old image tag
```

## Summary

Upgrading MySQL in Docker is safe when you follow a backup-first approach, use the same named volume across container versions, and watch the startup logs for upgrade confirmation. Minor version upgrades are generally automatic. For major version upgrades, run pre-upgrade compatibility checks and have a tested rollback plan using volume snapshots or SQL dumps.
