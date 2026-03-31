# How to Configure MySQL in Docker with Custom my.cnf

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Docker, Configuration, my.cnf, Performance

Description: Mount a custom my.cnf into a MySQL Docker container to override default settings for buffer pool size, connections, slow query logging, and character sets.

---

## How MySQL Docker Image Handles Configuration

The official MySQL Docker image reads configuration from `/etc/mysql/my.cnf` and any files matching `/etc/mysql/conf.d/*.cnf`. The cleanest approach is to mount your custom configuration into `/etc/mysql/conf.d/` so it supplements rather than replaces the base configuration.

## Creating a Custom Configuration File

Create `./mysql/my.cnf` on the host:

```ini
[mysqld]
# Memory settings
innodb_buffer_pool_size       = 1G
innodb_buffer_pool_instances  = 2
innodb_log_file_size          = 256M

# Connection settings
max_connections               = 200
wait_timeout                  = 600
interactive_timeout           = 600

# Slow query logging
slow_query_log                = 1
slow_query_log_file           = /var/log/mysql/slow.log
long_query_time               = 1
log_queries_not_using_indexes = 1

# Character set
character_set_server          = utf8mb4
collation_server              = utf8mb4_unicode_ci

# Networking
bind_address                  = 0.0.0.0
```

## Mounting with docker run

```bash
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=root_secret \
  -v $(pwd)/mysql/my.cnf:/etc/mysql/conf.d/custom.cnf:ro \
  -v mysql_data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8.0
```

The `:ro` flag mounts the file as read-only inside the container.

## Mounting with Docker Compose

```yaml
version: "3.9"

services:
  mysql:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root_secret
      MYSQL_DATABASE: myapp
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/my.cnf:/etc/mysql/conf.d/custom.cnf:ro
    ports:
      - "3306:3306"

volumes:
  mysql_data:
```

## Verifying the Configuration Was Applied

Connect to MySQL and check that your settings took effect:

```bash
docker exec -it mysql mysql -u root -proot_secret \
  -e "SELECT @@innodb_buffer_pool_size, @@max_connections, @@character_set_server;"
```

Or inspect a specific variable:

```bash
docker exec -it mysql mysql -u root -proot_secret \
  -e "SHOW VARIABLES LIKE 'slow_query_log%';"
```

## Passing Configuration as Command Arguments

For simple overrides you can pass `--variable=value` after the image name instead of mounting a file:

```bash
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=root_secret \
  mysql:8.0 \
  --innodb-buffer-pool-size=1073741824 \
  --max-connections=200
```

This works for a few variables but becomes unwieldy for more than two or three settings. Use a mounted `my.cnf` for anything beyond basic overrides.

## Troubleshooting Configuration Issues

If MySQL fails to start after mounting a custom config, check the logs:

```bash
docker logs mysql 2>&1 | tail -30
```

Common issues:
- **Unknown variable**: The variable name is misspelled or not valid for the MySQL version you are running.
- **Value out of range**: A numeric value exceeds the maximum allowed for the variable.
- **Permission denied**: The mounted file is owned by root and MySQL cannot read it. Fix with `chmod 644 ./mysql/my.cnf`.

## Summary

Mount a custom `my.cnf` into `/etc/mysql/conf.d/` as a read-only volume to tune MySQL running in Docker. This approach is non-destructive - it supplements the base configuration rather than replacing it. Verify your settings took effect using `SHOW VARIABLES` and check container logs if MySQL fails to start after a configuration change.
