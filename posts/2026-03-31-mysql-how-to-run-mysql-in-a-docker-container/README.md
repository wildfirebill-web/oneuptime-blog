# How to Run MySQL in a Docker Container

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Docker, Containers, DevOps, Development

Description: Run MySQL in a Docker container with persistent storage, custom configuration, and environment variable setup for development and production use.

---

## Quick Start

The fastest way to run MySQL in Docker:

```bash
docker run -d \
  --name mysql-server \
  -e MYSQL_ROOT_PASSWORD=rootpassword \
  -p 3306:3306 \
  mysql:8.0
```

Connect immediately:

```bash
docker exec -it mysql-server mysql -u root -prootpassword
```

## Running with Persistent Storage

Without a volume, all data is lost when the container is removed. Always use a volume in any environment:

```bash
docker run -d \
  --name mysql-server \
  -e MYSQL_ROOT_PASSWORD=rootpassword \
  -e MYSQL_DATABASE=myapp \
  -e MYSQL_USER=appuser \
  -e MYSQL_PASSWORD=apppassword \
  -v mysql-data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8.0
```

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `MYSQL_ROOT_PASSWORD` | Yes (or use ALLOW_EMPTY) | Root user password |
| `MYSQL_DATABASE` | No | Database to create on startup |
| `MYSQL_USER` | No | Additional user to create |
| `MYSQL_PASSWORD` | No | Password for `MYSQL_USER` |
| `MYSQL_ALLOW_EMPTY_PASSWORD` | No | Allow empty root password |

## Custom MySQL Configuration

Mount a custom `my.cnf` to override defaults:

```bash
# Create config directory
mkdir -p ~/mysql-config

cat > ~/mysql-config/my.cnf << 'EOF'
[mysqld]
max_connections=200
innodb_buffer_pool_size=512M
slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=1
character_set_server=utf8mb4
collation_server=utf8mb4_unicode_ci
EOF
```

```bash
docker run -d \
  --name mysql-server \
  -e MYSQL_ROOT_PASSWORD=rootpassword \
  -v mysql-data:/var/lib/mysql \
  -v ~/mysql-config/my.cnf:/etc/mysql/conf.d/custom.cnf:ro \
  -p 3306:3306 \
  mysql:8.0
```

## Using Docker Compose

For a more maintainable setup, use Docker Compose:

```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    container_name: mysql-server
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: myapp
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppassword
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init:/docker-entrypoint-initdb.d
      - ./config/my.cnf:/etc/mysql/conf.d/custom.cnf:ro

volumes:
  mysql-data:
```

```bash
docker compose up -d
```

## Running Initialization Scripts

Place `.sql` or `.sh` files in `/docker-entrypoint-initdb.d` to run them on first startup:

```sql
-- init/01-schema.sql
CREATE TABLE IF NOT EXISTS users (
    id   INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);
```

```bash
docker run -d \
  --name mysql-server \
  -e MYSQL_ROOT_PASSWORD=rootpassword \
  -e MYSQL_DATABASE=myapp \
  -v mysql-data:/var/lib/mysql \
  -v ./init:/docker-entrypoint-initdb.d \
  mysql:8.0
```

## Checking Container Health

```bash
# View logs
docker logs mysql-server

# Check container status
docker ps -a --filter name=mysql-server

# Run a health check query
docker exec mysql-server mysqladmin -u root -prootpassword ping
```

## Stopping and Removing

```bash
# Stop the container (data persists in volume)
docker stop mysql-server

# Remove container only (volume intact)
docker rm mysql-server

# Remove everything including data
docker rm -v mysql-server
docker volume rm mysql-data
```

## Summary

Running MySQL in Docker requires specifying a root password via environment variables and mounting a named volume at `/var/lib/mysql` to persist data across container restarts. Custom configuration is applied by mounting a `.cnf` file into `/etc/mysql/conf.d/`, and initialization scripts placed in `/docker-entrypoint-initdb.d/` run automatically on the first container startup.
