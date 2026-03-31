# How to Persist MySQL Data in Docker Volumes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Docker, Volume, Persistence, Container

Description: Configure Docker named volumes and bind mounts to persist MySQL data across container restarts, upgrades, and removals without losing your database.

---

## Why Data Persistence Matters in Docker

By default, files written inside a Docker container exist only for the container's lifetime. When you run `docker rm`, all MySQL data is lost. Docker volumes decouple the data lifecycle from the container lifecycle, allowing you to delete, recreate, or upgrade the MySQL container without affecting the data.

## Named Volumes (Recommended)

Named volumes are managed by Docker and stored under `/var/lib/docker/volumes/`. They are the recommended approach for database data.

```bash
# Create an explicit named volume
docker volume create mysql_data

# Run MySQL with the named volume
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=root_secret \
  -e MYSQL_DATABASE=myapp \
  -v mysql_data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8.0
```

With Docker Compose:

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root_secret
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
    driver: local
```

## Bind Mounts for Direct Host Access

Bind mounts map a specific host directory into the container. This gives you direct access to the MySQL data files from the host:

```bash
mkdir -p /data/mysql

docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=root_secret \
  -v /data/mysql:/var/lib/mysql \
  mysql:8.0
```

Bind mounts are useful when you want to back up data files using host-level tools, but they are more fragile because file permissions must match what MySQL expects (UID 999 in the official image).

Fix permissions if needed:

```bash
sudo chown -R 999:999 /data/mysql
```

## Verifying Persistence

Stop and remove the container, then start a new one with the same volume:

```bash
docker stop mysql && docker rm mysql

docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=root_secret \
  -v mysql_data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8.0

# Verify data survived
docker exec -it mysql mysql -u root -proot_secret -e "SHOW DATABASES;"
```

## Inspecting and Backing Up Volumes

```bash
# Inspect volume details and mount path on host
docker volume inspect mysql_data

# Create a backup of the volume contents
docker run --rm \
  -v mysql_data:/var/lib/mysql:ro \
  -v $(pwd):/backup \
  ubuntu \
  tar czf /backup/mysql_backup_$(date +%Y%m%d).tar.gz -C /var/lib/mysql .
```

## Storing Logs on a Separate Volume

Separate data and log volumes to make each independently manageable:

```yaml
services:
  mysql:
    image: mysql:8.0
    volumes:
      - mysql_data:/var/lib/mysql
      - mysql_logs:/var/log/mysql
    command: >
      --slow_query_log=1
      --slow_query_log_file=/var/log/mysql/slow.log

volumes:
  mysql_data:
  mysql_logs:
```

## Cleaning Up Unused Volumes

```bash
# List all volumes
docker volume ls

# Remove a specific volume (data is permanently deleted)
docker volume rm mysql_data

# Remove all unused volumes
docker volume prune
```

## Summary

Use Docker named volumes to persist MySQL data across container lifecycle events. Named volumes are Docker-managed, portable across environments, and easy to back up with a temporary container. Use bind mounts only when you need direct host-level access to data files. Always verify persistence by stopping, removing, and recreating the container before relying on a new deployment in production.
