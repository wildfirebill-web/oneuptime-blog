# How to Build Custom ClickHouse Docker Images

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Dockerfile, Custom Image, Build, Container

Description: Learn how to build custom ClickHouse Docker images that bundle configuration, dictionaries, and initialization scripts for repeatable deployments.

---

The official `clickhouse/clickhouse-server` image is a solid base, but building a custom image lets you bake in configuration, initialization SQL, custom dictionaries, and pre-installed plugins. This ensures every deployment starts from a consistent, fully configured state.

## Why Build a Custom Image

- Embed configuration files so no external mounts are required
- Include initialization SQL scripts that run on first startup
- Pre-install ClickHouse plugins or custom functions
- Control the exact version deployed across environments
- Simplify Kubernetes deployments with a self-contained image

## Project Structure

```bash
mkdir clickhouse-custom-image && cd clickhouse-custom-image
mkdir -p config/server config/users init-scripts
```

## Dockerfile

```dockerfile
FROM clickhouse/clickhouse-server:24.3

# Copy server configuration overrides
COPY config/server/ /etc/clickhouse-server/config.d/

# Copy user configuration overrides
COPY config/users/ /etc/clickhouse-server/users.d/

# Copy initialization scripts (run once on first startup)
COPY init-scripts/ /docker-entrypoint-initdb.d/

# Set appropriate ownership
RUN chown -R clickhouse:clickhouse /etc/clickhouse-server/config.d/ \
    && chown -R clickhouse:clickhouse /etc/clickhouse-server/users.d/

EXPOSE 8123 9000 9440
```

## Configuration Files

Create `config/server/performance.xml`:

```xml
<clickhouse>
    <max_server_memory_usage_to_ram_ratio>0.6</max_server_memory_usage_to_ram_ratio>
    <max_concurrent_queries>200</max_concurrent_queries>
    <logger>
        <level>warning</level>
    </logger>
</clickhouse>
```

Create `config/users/app_user.xml`:

```xml
<clickhouse>
    <users>
        <app>
            <password>changeme</password>
            <networks><ip>::/0</ip></networks>
            <profile>default</profile>
            <quota>default</quota>
        </app>
    </users>
</clickhouse>
```

## Initialization Scripts

Scripts in `/docker-entrypoint-initdb.d/` are executed on the first container start. Create `init-scripts/01_create_schema.sql`:

```sql
CREATE DATABASE IF NOT EXISTS app;

CREATE TABLE IF NOT EXISTS app.events (
    event_id   UUID DEFAULT generateUUIDv4(),
    event_type LowCardinality(String),
    user_id    UInt64,
    created_at DateTime DEFAULT now()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id);
```

## Building the Image

```bash
docker build -t myorg/clickhouse:24.3-custom .
```

Verify the image was built:

```bash
docker images myorg/clickhouse
```

## Running the Custom Image

```bash
docker run -d \
  --name clickhouse \
  -p 8123:8123 \
  -p 9000:9000 \
  -v clickhouse_data:/var/lib/clickhouse \
  myorg/clickhouse:24.3-custom
```

## Checking Initialization Logs

```bash
docker logs clickhouse 2>&1 | grep -E "(init|schema|error|warning)"
```

## Tagging and Pushing to a Registry

```bash
docker tag myorg/clickhouse:24.3-custom registry.mycompany.com/clickhouse:24.3-custom
docker push registry.mycompany.com/clickhouse:24.3-custom
```

## Summary

Building custom ClickHouse Docker images produces self-contained, reproducible deployments. By embedding configuration and initialization scripts into the image, you eliminate external config management overhead and ensure every environment - dev, staging, and production - starts from the same known state.
