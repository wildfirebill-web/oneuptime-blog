# How to Set Up Single-Node ClickHouse with Docker Compose

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Docker Compose, Single Node, Setup, Self-Hosted

Description: Step-by-step guide to running a single-node ClickHouse instance using Docker Compose for local development and small-scale deployments.

---

Docker Compose is the fastest way to get a single-node ClickHouse instance running locally. This guide walks through a minimal setup and covers the key configuration options for development and testing workflows.

## Prerequisites

- Docker and Docker Compose installed
- At least 4 GB RAM available for ClickHouse

## Creating the Docker Compose File

Create a project directory and a `docker-compose.yml` file:

```bash
mkdir clickhouse-single && cd clickhouse-single
```

```yaml
version: "3.8"

services:
  clickhouse:
    image: clickhouse/clickhouse-server:24.3
    container_name: clickhouse
    ports:
      - "8123:8123"   # HTTP interface
      - "9000:9000"   # Native TCP interface
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - clickhouse_logs:/var/log/clickhouse-server
    environment:
      CLICKHOUSE_DB: default
      CLICKHOUSE_USER: admin
      CLICKHOUSE_PASSWORD: strongpassword
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
```

Verify the container is running:

```bash
docker compose ps
docker compose logs clickhouse
```

## Connecting to ClickHouse

Connect using the HTTP interface with curl:

```bash
curl -u admin:strongpassword \
  'http://localhost:8123/?query=SELECT+version()'
```

Or use the native client inside the container:

```bash
docker exec -it clickhouse clickhouse-client \
  --user admin --password strongpassword
```

Once connected, run a test query:

```sql
SELECT
    name,
    value
FROM system.settings
WHERE name IN ('max_memory_usage', 'max_threads')
LIMIT 10;
```

## Creating Your First Table

```sql
CREATE DATABASE analytics;

CREATE TABLE analytics.page_views (
    session_id  String,
    user_id     UInt64,
    page_url    String,
    viewed_at   DateTime DEFAULT now(),
    duration_ms UInt32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(viewed_at)
ORDER BY (viewed_at, user_id);

INSERT INTO analytics.page_views VALUES
    ('sess1', 1, '/home', now(), 1200),
    ('sess2', 2, '/pricing', now(), 3400);

SELECT * FROM analytics.page_views;
```

## Configuring Memory Limits

For development machines with limited RAM, add a custom configuration:

```bash
mkdir -p config
```

Create `config/custom.xml`:

```xml
<clickhouse>
    <max_server_memory_usage_to_ram_ratio>0.5</max_server_memory_usage_to_ram_ratio>
    <max_concurrent_queries>50</max_concurrent_queries>
</clickhouse>
```

Mount it in `docker-compose.yml`:

```yaml
volumes:
  - ./config/custom.xml:/etc/clickhouse-server/config.d/custom.xml
```

## Stopping and Cleaning Up

```bash
# Stop and remove containers (keeps data volumes)
docker compose down

# Remove everything including volumes
docker compose down -v
```

## Summary

A single-node ClickHouse setup with Docker Compose takes just a few minutes to configure and gives you a full-featured ClickHouse instance suitable for local development, testing, and small workloads. Use named volumes to persist data across container restarts.
