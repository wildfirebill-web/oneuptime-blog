# How to Deploy MySQL via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, MySQL, Database, Self-Hosted

Description: Deploy MySQL via Portainer with persistent storage, secure configuration, and optional PHPMyAdmin for web-based database management.

## Introduction

MySQL is the world's most popular open-source relational database. Deploying it via Portainer ensures persistent data storage, easy configuration management, and a clear upgrade path. This guide covers deploying MySQL with PHPMyAdmin for web-based administration.

## Deploy as a Stack

In Portainer, create a stack named `mysql`:

```yaml
version: "3.8"

services:
  mysql:
    image: mysql:8.4
    container_name: mysql
    environment:
      # Root password - change this!
      MYSQL_ROOT_PASSWORD: change_this_root_password
      # Create initial database and user
      MYSQL_DATABASE: myapp
      MYSQL_USER: myapp_user
      MYSQL_PASSWORD: change_this_app_password
    volumes:
      # Persistent data storage
      - mysql_data:/var/lib/mysql
      # Custom configuration
      - ./my.cnf:/etc/mysql/conf.d/custom.cnf:ro
      # Initialization scripts (run once on first start)
      - ./init:/docker-entrypoint-initdb.d:ro
    ports:
      - "3306:3306"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  # PHPMyAdmin - web-based MySQL administration
  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpmyadmin
    environment:
      PMA_HOST: mysql
      PMA_PORT: 3306
      PMA_USER: root
      PMA_PASSWORD: change_this_root_password
    ports:
      - "8080:80"
    depends_on:
      mysql:
        condition: service_healthy
    restart: unless-stopped

volumes:
  mysql_data:
    driver: local
```

## Custom MySQL Configuration

Create `my.cnf`:

```ini
[mysqld]
# Character set
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# Performance settings
innodb_buffer_pool_size = 256M
innodb_log_file_size = 64M
max_connections = 100

# Slow query log
slow_query_log = 1
slow_query_log_file = /var/lib/mysql/slow.log
long_query_time = 2

# Binary logging for replication
log_bin = /var/lib/mysql/mysql-bin
binlog_format = ROW
expire_logs_days = 7

# Security
sql_mode = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

## Initialization Scripts

Create `init/01-setup.sql` to run on first container start:

```sql
-- Create additional databases
CREATE DATABASE IF NOT EXISTS analytics CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE IF NOT EXISTS staging CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create additional users
CREATE USER IF NOT EXISTS 'analytics_user'@'%' IDENTIFIED BY 'analytics_password';
GRANT ALL PRIVILEGES ON analytics.* TO 'analytics_user'@'%';

-- Create read-only user for reporting
CREATE USER IF NOT EXISTS 'readonly_user'@'%' IDENTIFIED BY 'readonly_password';
GRANT SELECT ON myapp.* TO 'readonly_user'@'%';

FLUSH PRIVILEGES;
```

## Connecting Applications to MySQL

Other containers in the same Docker network can connect using the service name:

```yaml
version: "3.8"

services:
  webapp:
    image: myapp:latest
    environment:
      # Use MySQL container name as host
      DB_HOST: mysql
      DB_PORT: 3306
      DB_NAME: myapp
      DB_USER: myapp_user
      DB_PASSWORD: change_this_app_password
    depends_on:
      mysql:
        condition: service_healthy
```

## Backup and Restore

### Backup via Portainer Console

```bash
# Access MySQL container console in Portainer
# Or via docker exec:
docker exec mysql mysqldump \
  -u root -p"change_this_root_password" \
  --all-databases \
  --single-transaction \
  --flush-logs \
  > /tmp/mysql-backup-$(date +%Y%m%d).sql
```

### Restore from Backup

```bash
docker exec -i mysql mysql \
  -u root -p"change_this_root_password" \
  < mysql-backup.sql
```

## Security Hardening

After initial setup, remove test databases:

```sql
-- Run in PHPMyAdmin or MySQL console
DELETE FROM mysql.user WHERE User='';
DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1');
DROP DATABASE IF EXISTS test;
FLUSH PRIVILEGES;
```

## Conclusion

MySQL deployed via Portainer with persistent volumes and a custom configuration provides a reliable database backend for applications. PHPMyAdmin gives you web-based management without needing to install database clients. The healthcheck configuration ensures dependent applications wait for MySQL to be fully ready before attempting connections.
