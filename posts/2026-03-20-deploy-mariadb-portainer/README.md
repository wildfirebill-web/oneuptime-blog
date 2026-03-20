# How to Deploy MariaDB via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, MariaDB, Database, Docker, Deployment

Description: Learn how to deploy MariaDB via Portainer with persistent storage, configuration tuning, and connection from application containers.

## MariaDB vs MySQL

MariaDB is a MySQL-compatible database with active community development. It's drop-in compatible with MySQL for most use cases. Docker image: `mariadb:latest` or `mariadb:11`.

## Deploy via Portainer Stack

**Stacks → Add Stack → mariadb**

```yaml
version: "3.8"

services:
  mariadb:
    image: mariadb:11
    restart: unless-stopped
    environment:
      - MARIADB_ROOT_PASSWORD=${MARIADB_ROOT_PASSWORD}
      - MARIADB_DATABASE=${MARIADB_DATABASE}
      - MARIADB_USER=${MARIADB_USER}
      - MARIADB_PASSWORD=${MARIADB_PASSWORD}
      # Optional: root host restriction
      - MARIADB_ROOT_HOST=%
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./mariadb-conf:/etc/mysql/conf.d:ro
    ports:
      - "127.0.0.1:3306:3306"
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--su-mysql", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mariadb_data:
```

## MariaDB Configuration

```ini
# mariadb-conf/server.cnf
[mysqld]
# Performance
innodb_buffer_pool_size = 256M
innodb_log_file_size = 64M
innodb_flush_method = O_DIRECT

# Character set
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# Connections
max_connections = 150
max_allowed_packet = 64M

# Slow queries
slow_query_log = 1
long_query_time = 1

# Binary log (needed for replication and point-in-time recovery)
log_bin = /var/lib/mysql/mysql-bin
expire_logs_days = 7
max_binlog_size = 100M
```

## Application Stack with MariaDB

```yaml
version: "3.8"

services:
  app:
    image: wordpress:latest
    restart: unless-stopped
    ports:
      - "8080:80"
    environment:
      - WORDPRESS_DB_HOST=mariadb:3306
      - WORDPRESS_DB_USER=${MARIADB_USER}
      - WORDPRESS_DB_PASSWORD=${MARIADB_PASSWORD}
      - WORDPRESS_DB_NAME=${MARIADB_DATABASE}
    depends_on:
      mariadb:
        condition: service_healthy

  mariadb:
    image: mariadb:11
    restart: unless-stopped
    environment:
      - MARIADB_ROOT_PASSWORD=${MARIADB_ROOT_PASSWORD}
      - MARIADB_DATABASE=${MARIADB_DATABASE}
      - MARIADB_USER=${MARIADB_USER}
      - MARIADB_PASSWORD=${MARIADB_PASSWORD}
    volumes:
      - mariadb_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--su-mysql", "--connect"]
      interval: 10s
      retries: 5

volumes:
  mariadb_data:
```

## Creating Databases and Users

Via Portainer console:

```sql
-- Connect as root
mysql -u root -p"${MARIADB_ROOT_PASSWORD}"

-- Create additional database
CREATE DATABASE analytics CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON analytics.* TO 'analytics_user'@'%' IDENTIFIED BY 'secure_password';
FLUSH PRIVILEGES;

-- Check databases
SHOW DATABASES;
```

## MariaDB Backup

```bash
# Backup all databases
docker exec mariadb mysqldump \
  -u root -p"${MARIADB_ROOT_PASSWORD}" \
  --all-databases \
  --single-transaction \
  --quick \
  > /backup/mariadb-$(date +%Y%m%d-%H%M%S).sql

# Restore
docker exec -i mariadb mysql \
  -u root -p"${MARIADB_ROOT_PASSWORD}" \
  < /backup/mariadb-20260320.sql
```

## Upgrading MariaDB

To upgrade from MariaDB 10.x to 11:

1. Back up all data
2. Update the image tag in Portainer: `mariadb:10` → `mariadb:11`
3. Update the stack — MariaDB handles in-place upgrade automatically
4. Verify: `docker exec mariadb mariadb --version`

## Conclusion

MariaDB via Portainer is nearly identical to MySQL in setup and management. The improved healthcheck command (`healthcheck.sh --su-mysql`) is more reliable than MySQL's equivalent. As with any database deployment, persistent volumes and environment-variable-managed credentials are non-negotiable for safe operations.
