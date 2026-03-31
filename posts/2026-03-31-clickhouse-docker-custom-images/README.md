# How to Build Custom ClickHouse Docker Images

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Custom Image, Dockerfile, Container, Build

Description: Learn how to build custom ClickHouse Docker images that include pre-baked configuration, initialization scripts, and custom plugins for reproducible deployments.

---

The official `clickhouse/clickhouse-server` Docker image is a solid base, but building custom images lets you bake in configuration, seed data, initialization SQL scripts, and custom dictionaries - making deployments fully self-contained and reproducible.

## Basic Custom Dockerfile

```dockerfile
FROM clickhouse/clickhouse-server:24.3

# Copy custom server configuration
COPY config/memory.xml /etc/clickhouse-server/config.d/memory.xml
COPY config/query-limits.xml /etc/clickhouse-server/config.d/query-limits.xml

# Copy custom user configuration
COPY config/users.xml /etc/clickhouse-server/users.d/users.xml

# Copy initialization SQL scripts
COPY init-scripts/ /docker-entrypoint-initdb.d/
```

## Initialization Scripts

Any `.sql` or `.sh` files placed in `/docker-entrypoint-initdb.d/` are executed when the container starts for the first time:

`init-scripts/01-create-schema.sql`:
```sql
CREATE DATABASE IF NOT EXISTS analytics;

CREATE TABLE IF NOT EXISTS analytics.events (
    event_id UInt64,
    event_type String,
    user_id UInt64,
    event_date Date DEFAULT today(),
    ts DateTime DEFAULT now()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, event_id);
```

`init-scripts/02-seed-data.sh`:
```bash
#!/bin/bash
clickhouse-client --query "INSERT INTO analytics.events VALUES (1, 'click', 100, today(), now())"
```

## Including Custom Dictionaries

```dockerfile
FROM clickhouse/clickhouse-server:24.3

COPY dictionaries/ /etc/clickhouse-server/dictionaries/
COPY config/dictionaries.xml /etc/clickhouse-server/config.d/dictionaries.xml
```

`config/dictionaries.xml`:
```xml
<clickhouse>
  <dictionaries_config>/etc/clickhouse-server/dictionaries/*.xml</dictionaries_config>
</clickhouse>
```

## Building the Image

```bash
docker build -t my-company/clickhouse:24.3 .

# Test it
docker run --rm my-company/clickhouse:24.3 clickhouse-client --query "SELECT version()"
```

## Multi-Stage Build for Config Validation

```dockerfile
FROM clickhouse/clickhouse-server:24.3 AS validator
COPY config/ /etc/clickhouse-server/config.d/
RUN clickhouse-server --config-check

FROM clickhouse/clickhouse-server:24.3
COPY --from=validator /etc/clickhouse-server/config.d/ /etc/clickhouse-server/config.d/
COPY init-scripts/ /docker-entrypoint-initdb.d/
```

## Pushing to a Registry

```bash
docker tag my-company/clickhouse:24.3 registry.example.com/clickhouse:24.3
docker push registry.example.com/clickhouse:24.3
```

## Summary

Custom ClickHouse Docker images embed configuration, initialization SQL scripts, and dictionaries directly into the image for fully self-contained deployments. Use `/docker-entrypoint-initdb.d/` for startup scripts, layer in config files via `COPY`, and validate configuration with a multi-stage build to catch errors before pushing to your registry.
