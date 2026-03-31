# How to Set Up a MySQL Test Database with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Docker, Testing, Configuration

Description: Learn how to set up a fast, isolated MySQL test database using Docker, including schema initialization, tmpfs storage, health checks, and CI integration.

---

## Why Docker for a MySQL Test Database

Running tests against a local or shared MySQL instance causes test pollution and environment inconsistencies. A Docker container gives each developer or CI job an isolated, ephemeral MySQL instance that starts clean every time and matches the production MySQL version exactly.

## Quickstart with Docker Run

```bash
docker run --name mysql-test \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=app_test \
  -e MYSQL_USER=testuser \
  -e MYSQL_PASSWORD=testpass \
  -p 3307:3306 \
  --tmpfs /var/lib/mysql \
  -d mysql:8.0

# Wait until healthy
until docker exec mysql-test mysqladmin ping -h localhost -u root -prootpass --silent; do
  sleep 1
done
echo "MySQL is ready"
```

The `--tmpfs` flag stores data in memory, removing I/O overhead and making the container disposable.

## Docker Compose Setup

For projects with multiple services, use Docker Compose:

```yaml
version: "3.8"
services:
  mysql-test:
    image: mysql:8.0
    container_name: mysql-test
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: app_test
      MYSQL_USER: testuser
      MYSQL_PASSWORD: testpass
    ports:
      - "3307:3306"
    tmpfs:
      - /var/lib/mysql
    volumes:
      - ./docker/mysql/init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-prootpass"]
      interval: 5s
      timeout: 3s
      retries: 12
```

## Auto-Initializing Schema

MySQL's Docker image executes all `.sql` and `.sh` files in `/docker-entrypoint-initdb.d` on first start. Place your schema and seed files there:

```bash
mkdir -p docker/mysql/init
```

```sql
-- docker/mysql/init/01_schema.sql
CREATE TABLE IF NOT EXISTS users (
  id         BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  email      VARCHAR(200) NOT NULL UNIQUE,
  created_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS orders (
  id          BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
  user_id     BIGINT        NOT NULL,
  total       DECIMAL(10,2) NOT NULL,
  status      VARCHAR(20)   NOT NULL DEFAULT 'PENDING',
  created_at  TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user (user_id)
) ENGINE=InnoDB;
```

```sql
-- docker/mysql/init/02_seed.sql
INSERT INTO users (email) VALUES
  ('alice@test.com'),
  ('bob@test.com');
```

## Custom MySQL Configuration for Tests

Override MySQL settings to optimize for test speed, not durability:

```ini
# docker/mysql/conf/test.cnf
[mysqld]
# Disable binary log - not needed for tests
skip-log-bin
# Faster commits - safe to lose data in test container
innodb_flush_log_at_trx_commit = 0
sync_binlog                    = 0
# Reduce memory footprint
innodb_buffer_pool_size        = 256M
```

Mount the config in Docker Compose:

```yaml
    volumes:
      - ./docker/mysql/init:/docker-entrypoint-initdb.d
      - ./docker/mysql/conf/test.cnf:/etc/mysql/conf.d/test.cnf
```

## Connect and Verify

```bash
mysql -h 127.0.0.1 -P 3307 -u testuser -ptestpass app_test \
  -e "SHOW TABLES;"
```

## Teardown

```bash
# Stop and remove container (data is gone since tmpfs was used)
docker compose -f docker-compose.test.yml down

# Or if using docker run
docker stop mysql-test && docker rm mysql-test
```

## GitHub Actions Integration

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: rootpass
          MYSQL_DATABASE: app_test
          MYSQL_USER: testuser
          MYSQL_PASSWORD: testpass
        ports:
          - 3307:3306
        options: >-
          --health-cmd="mysqladmin ping -prootpass"
          --health-interval=5s
          --health-timeout=3s
          --health-retries=10
    steps:
      - uses: actions/checkout@v4
      - name: Run schema migrations
        run: |
          mysql -h 127.0.0.1 -P 3307 -u testuser -ptestpass app_test \
            < docker/mysql/init/01_schema.sql
      - name: Run tests
        run: npm test
        env:
          DB_HOST: 127.0.0.1
          DB_PORT: 3307
          DB_NAME: app_test
          DB_USER: testuser
          DB_PASS: testpass
```

## Summary

Docker provides an ideal test database environment: isolated, reproducible, and disposable. Use `tmpfs` for fast in-memory storage, place initialization SQL files in the init directory, override MySQL config to trade durability for speed, and integrate with CI using service containers. This setup gives every developer and pipeline a consistent MySQL environment from a single configuration file.
