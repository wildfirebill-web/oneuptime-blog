# How to Persist ClickHouse Data with Docker Volumes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Docker Volume, Persistent Storage, Data Management, Container

Description: Learn how to configure Docker volumes for ClickHouse containers to ensure data persistence across container restarts, upgrades, and re-deployments.

---

By default, data inside a Docker container is lost when the container is removed. For ClickHouse, you must mount volumes to persist your data, logs, and configuration across container lifecycle events.

## Key Directories to Persist

| Directory | Purpose |
|---|---|
| `/var/lib/clickhouse` | All table data, metadata, and parts |
| `/var/log/clickhouse-server` | Server logs |
| `/etc/clickhouse-server/config.d` | Custom config files |
| `/etc/clickhouse-server/users.d` | Custom user configs |

## Named Volumes (Recommended)

Named volumes are managed by Docker and survive `docker compose down` without `-v`:

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - clickhouse_logs:/var/log/clickhouse-server

volumes:
  clickhouse_data:
    driver: local
  clickhouse_logs:
    driver: local
```

## Bind Mounts (Host Directory)

Bind mounts map a specific host path into the container, useful when you need direct host access to data:

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    volumes:
      - /data/clickhouse:/var/lib/clickhouse
      - /logs/clickhouse:/var/log/clickhouse-server
      - ./config:/etc/clickhouse-server/config.d
```

## Checking Volume Contents

```bash
# Inspect a named volume
docker volume inspect clickhouse_data

# List files in the data directory
docker exec clickhouse ls -la /var/lib/clickhouse/
```

## Verifying Data Persistence

```bash
# Create test data
docker exec clickhouse clickhouse-client \
  --query "CREATE DATABASE IF NOT EXISTS test;
           CREATE TABLE IF NOT EXISTS test.t (id UInt64) ENGINE=MergeTree ORDER BY id;
           INSERT INTO test.t VALUES (1),(2),(3)"

# Stop and remove the container (keep volumes)
docker compose down

# Start again
docker compose up -d

# Verify data survived
docker exec clickhouse clickhouse-client \
  --query "SELECT * FROM test.t"
```

## Upgrading ClickHouse with Existing Volumes

```bash
# Pull the new image
docker compose pull

# Restart with the new image (data volumes remain unchanged)
docker compose up -d
```

ClickHouse automatically handles data format migrations when the version changes.

## Backing Up Volume Data

```bash
docker run --rm \
  -v clickhouse_data:/source \
  -v $(pwd)/backup:/backup \
  alpine tar czf /backup/clickhouse_data.tar.gz -C /source .
```

## Summary

Always use named volumes or bind mounts for `/var/lib/clickhouse` to persist ClickHouse data across container restarts and upgrades. Named volumes are recommended for production as Docker manages their lifecycle independently from containers, preventing accidental data loss during `docker compose down`.
