# How to Set Up Single-Node ClickHouse with Docker Compose

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Docker Compose, Single-Node, Local Development, Setup

Description: Learn how to run a single-node ClickHouse instance using Docker Compose for local development and testing with persistent storage and proper configuration.

---

Docker Compose is the fastest way to get a ClickHouse instance running locally. A single-node setup is ideal for development, CI/CD environments, and learning ClickHouse without the complexity of a full cluster.

## Basic docker-compose.yml

```yaml
version: '3.8'

services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    container_name: clickhouse
    ports:
      - "8123:8123"   # HTTP interface
      - "9000:9000"   # Native protocol
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - clickhouse_logs:/var/log/clickhouse-server
    environment:
      CLICKHOUSE_DB: default
      CLICKHOUSE_USER: default
      CLICKHOUSE_PASSWORD: clickhouse
      CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT: 1
    ulimits:
      nofile:
        soft: 262144
        hard: 262144

volumes:
  clickhouse_data:
  clickhouse_logs:
```

## Starting the Container

```bash
docker compose up -d
docker compose logs -f clickhouse
```

Wait for the log line `Application: Ready for connections` before connecting.

## Connecting to ClickHouse

Using the native client:

```bash
docker exec -it clickhouse clickhouse-client --user default --password clickhouse
```

Using `curl` against the HTTP interface:

```bash
curl -u default:clickhouse 'http://localhost:8123/?query=SELECT+version()'
```

## Verifying the Setup

```sql
SELECT version();
SELECT * FROM system.databases;
CREATE DATABASE IF NOT EXISTS test;
USE test;
CREATE TABLE events (
    id UInt64,
    name String,
    ts DateTime DEFAULT now()
) ENGINE = MergeTree()
ORDER BY id;
INSERT INTO events VALUES (1, 'test_event', now());
SELECT * FROM events;
```

## Adding a Health Check

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8123/ping"]
      interval: 10s
      timeout: 5s
      retries: 5
```

## Stopping and Removing

```bash
# Stop but keep data
docker compose stop

# Remove containers (data volumes persist)
docker compose down

# Remove everything including volumes
docker compose down -v
```

## Summary

A single-node ClickHouse Docker Compose setup requires just a few lines of YAML. Use named volumes for persistent storage, expose both the HTTP (8123) and native (9000) ports, and set `CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT=1` to enable SQL-based user management. Add a health check to ensure dependent services wait for ClickHouse to be ready.
