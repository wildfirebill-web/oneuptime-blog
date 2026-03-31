# How to Use Custom ClickHouse Configuration with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Configuration, config.d, users.d, Container

Description: Learn how to apply custom ClickHouse server and user configuration when running in Docker by mounting config files into the config.d and users.d directories.

---

ClickHouse supports a layered configuration system where files placed in `config.d/` and `users.d/` directories are merged with the base configuration. This makes it easy to apply custom settings in Docker without replacing the default config files.

## Configuration Directory Structure

```text
/etc/clickhouse-server/
    config.xml           (base config - do not modify directly)
    config.d/            (drop-in overrides for server config)
        my-settings.xml
    users.xml            (base users config)
    users.d/             (drop-in overrides for users)
        my-users.xml
```

## Custom Memory Settings

Create `config/memory.xml`:

```xml
<clickhouse>
  <max_server_memory_usage_to_ram_ratio>0.8</max_server_memory_usage_to_ram_ratio>
  <max_memory_usage>10000000000</max_memory_usage>
</clickhouse>
```

## Custom Query Limits

Create `config/query-limits.xml`:

```xml
<clickhouse>
  <profiles>
    <default>
      <max_execution_time>60</max_execution_time>
      <max_rows_to_read>1000000000</max_rows_to_read>
      <max_bytes_to_read>100000000000</max_bytes_to_read>
    </default>
  </profiles>
</clickhouse>
```

## Custom User Configuration

Create `config/users.xml` (place in `users.d/`):

```xml
<clickhouse>
  <users>
    <analyst>
      <password_sha256_hex>your-sha256-hash</password_sha256_hex>
      <networks>
        <ip>::/0</ip>
      </networks>
      <profile>default</profile>
      <quota>default</quota>
      <allow_databases>
        <database>analytics</database>
      </allow_databases>
    </analyst>
  </users>
</clickhouse>
```

## Docker Compose with Custom Config

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - ./config/memory.xml:/etc/clickhouse-server/config.d/memory.xml
      - ./config/query-limits.xml:/etc/clickhouse-server/config.d/query-limits.xml
      - ./config/analyst-user.xml:/etc/clickhouse-server/users.d/analyst-user.xml

volumes:
  clickhouse_data:
```

## Using Environment Variables for Simple Settings

For simple settings, ClickHouse Docker image supports environment variables:

```yaml
environment:
  CLICKHOUSE_DB: analytics
  CLICKHOUSE_USER: default
  CLICKHOUSE_PASSWORD: securepass
  CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT: 1
```

## Verifying Configuration Was Applied

```sql
SELECT name, value
FROM system.settings
WHERE name IN (
  'max_execution_time',
  'max_memory_usage',
  'max_rows_to_read'
);
```

## Summary

Apply custom ClickHouse configuration in Docker by mounting XML files into `/etc/clickhouse-server/config.d/` for server settings and `/etc/clickhouse-server/users.d/` for user configuration. The layered merge system means you only need to specify the settings you want to override, leaving all defaults intact.
