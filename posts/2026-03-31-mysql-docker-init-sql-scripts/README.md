# How to Initialize MySQL in Docker with SQL Scripts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Docker, Initialization, SQL, Schema

Description: Use Docker's /docker-entrypoint-initdb.d directory to automatically run SQL scripts when a MySQL container is first created, setting up schemas and seed data.

---

## How MySQL Docker Initialization Works

The official MySQL Docker image checks the `/docker-entrypoint-initdb.d/` directory on first startup. Any `.sql`, `.sql.gz`, or `.sh` files found there are executed in alphabetical order against the default database (set by `MYSQL_DATABASE`). This happens only when the data directory (`/var/lib/mysql`) is empty - on subsequent container starts the scripts are skipped.

## Creating Initialization Scripts

Create a directory on the host to hold your scripts:

```bash
mkdir -p ./mysql/initdb
```

Create `./mysql/initdb/01-schema.sql`:

```sql
CREATE TABLE IF NOT EXISTS users (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  email       VARCHAR(255) NOT NULL UNIQUE,
  name        VARCHAR(255) NOT NULL,
  created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS products (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name        VARCHAR(255) NOT NULL,
  price       DECIMAL(10,2) NOT NULL,
  stock       INT NOT NULL DEFAULT 0,
  created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Create `./mysql/initdb/02-seed.sql`:

```sql
INSERT INTO users (email, name) VALUES
  ('alice@example.com', 'Alice Smith'),
  ('bob@example.com', 'Bob Jones');

INSERT INTO products (name, price, stock) VALUES
  ('Widget A', 9.99, 100),
  ('Widget B', 19.99, 50);
```

## Running with docker run

```bash
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=root_secret \
  -e MYSQL_DATABASE=myapp \
  -v $(pwd)/mysql/initdb:/docker-entrypoint-initdb.d:ro \
  -v mysql_data:/var/lib/mysql \
  mysql:8.0
```

## Using Docker Compose

```yaml
version: "3.9"

services:
  mysql:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root_secret
      MYSQL_DATABASE: myapp
      MYSQL_USER: appuser
      MYSQL_PASSWORD: app_secret
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/initdb:/docker-entrypoint-initdb.d:ro

volumes:
  mysql_data:
```

## Using Shell Scripts for Complex Setup

For logic that SQL alone cannot handle, use a `.sh` script:

```bash
# ./mysql/initdb/03-create-users.sh
#!/bin/bash
mysql -u root -p"$MYSQL_ROOT_PASSWORD" <<-EOF
CREATE USER IF NOT EXISTS 'readonly_user'@'%'
  IDENTIFIED BY 'readonly_secret';
GRANT SELECT ON myapp.* TO 'readonly_user'@'%';
FLUSH PRIVILEGES;
EOF
```

Shell scripts in `docker-entrypoint-initdb.d/` have access to the `MYSQL_ROOT_PASSWORD` environment variable.

## Forcing Re-initialization

Because initialization only runs when the data directory is empty, you must remove the volume to re-run the scripts:

```bash
docker compose down -v
docker compose up -d
```

This destroys all data. In production, use a migration tool like Flyway or Liquibase instead of init scripts.

## Verifying Initialization

```bash
docker exec -it mysql mysql -u root -proot_secret myapp \
  -e "SHOW TABLES; SELECT * FROM users;"
```

## Summary

MySQL Docker init scripts placed in `/docker-entrypoint-initdb.d/` execute automatically on the first container start, making them ideal for development and CI environments that need a pre-populated schema. Name files with numeric prefixes to control execution order, and use `.sh` scripts when you need conditional logic or user management beyond plain SQL. For production schema management, prefer migration tools over init scripts.
