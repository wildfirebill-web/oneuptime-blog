# How to Use Custom ClickHouse Configuration with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Configuration, XML, Override, Tuning

Description: Learn how to supply custom ClickHouse configuration files when running in Docker, including memory limits, user settings, and cluster definitions.

---

ClickHouse uses XML-based configuration files. When running in Docker, you can inject custom configuration by mounting files into the `/etc/clickhouse-server/config.d/` and `/etc/clickhouse-server/users.d/` override directories. Files placed there are merged with the defaults, letting you tune specific settings without replacing the entire config.

## Configuration File Locations

| Directory | Purpose |
|-----------|---------|
| `/etc/clickhouse-server/config.xml` | Main server configuration (do not replace) |
| `/etc/clickhouse-server/config.d/` | Override files merged into config.xml |
| `/etc/clickhouse-server/users.xml` | Default user configuration |
| `/etc/clickhouse-server/users.d/` | Override files merged into users.xml |

## Project Structure

```bash
mkdir -p clickhouse-custom/config/server
mkdir -p clickhouse-custom/config/users
```

## Server Configuration Overrides

Create `config/server/tuning.xml` to adjust memory and query settings:

```xml
<clickhouse>
    <!-- Use at most 50% of available RAM -->
    <max_server_memory_usage_to_ram_ratio>0.5</max_server_memory_usage_to_ram_ratio>

    <!-- Allow up to 100 concurrent queries -->
    <max_concurrent_queries>100</max_concurrent_queries>

    <!-- Background merge threads -->
    <background_pool_size>16</background_pool_size>

    <!-- Logging level -->
    <logger>
        <level>warning</level>
    </logger>
</clickhouse>
```

## User Configuration Overrides

Create `config/users/custom_users.xml` to define a non-default user:

```xml
<clickhouse>
    <users>
        <analytics_user>
            <password_sha256_hex>your_hashed_password_here</password_sha256_hex>
            <networks>
                <ip>::/0</ip>
            </networks>
            <profile>default</profile>
            <quota>default</quota>
            <allow_databases>
                <database>analytics</database>
            </allow_databases>
        </analytics_user>
    </users>
</clickhouse>
```

Generate a SHA256 password hash:

```bash
echo -n "mypassword" | sha256sum | tr -d ' -'
```

## Docker Compose with Custom Config

```yaml
version: "3.8"

services:
  clickhouse:
    image: clickhouse/clickhouse-server:24.3
    container_name: clickhouse
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - ./config/server:/etc/clickhouse-server/config.d
      - ./config/users:/etc/clickhouse-server/users.d
    ulimits:
      nofile:
        soft: 262144
        hard: 262144

volumes:
  clickhouse_data:
```

## Applying ZooKeeper Configuration

For clustered setups, add a ZooKeeper config in `config/server/zookeeper.xml`:

```xml
<clickhouse>
    <zookeeper>
        <node>
            <host>zookeeper</host>
            <port>2181</port>
        </node>
    </zookeeper>
</clickhouse>
```

## Verifying Configuration Was Applied

After starting the container, verify the settings:

```bash
docker exec -it clickhouse clickhouse-client \
  --query "SELECT name, value FROM system.server_settings WHERE name LIKE 'max_%' LIMIT 10;"
```

Check that user was created:

```bash
docker exec -it clickhouse clickhouse-client \
  --query "SELECT name, storage FROM system.users;"
```

## Reloading Config Without Restart

ClickHouse supports live config reload for some settings:

```bash
docker exec -it clickhouse clickhouse-client \
  --query "SYSTEM RELOAD CONFIG;"
```

## Summary

Mounting custom XML overrides into ClickHouse's config.d and users.d directories is the cleanest way to customize a Dockerized ClickHouse instance. This approach keeps your changes isolated, version-controllable, and compatible with future image updates without replacing base configuration files.
