# How to Manage ClickHouse Docker Container Logs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Logging, Log Management, Troubleshooting, Monitoring

Description: Learn how to access, configure, and manage ClickHouse logs in Docker containers including log rotation, level configuration, and forwarding to centralized log systems.

---

ClickHouse produces several types of logs: server logs, query logs, and trace logs. When running in Docker, managing these effectively prevents disk exhaustion and makes troubleshooting much easier.

## Accessing Logs

```bash
# View live server logs
docker logs -f clickhouse-dev

# Last 100 lines
docker logs --tail 100 clickhouse-dev

# Logs with timestamps
docker logs --timestamps clickhouse-dev
```

## ClickHouse Log Files Inside the Container

```bash
# List log files
docker exec clickhouse ls -la /var/log/clickhouse-server/

# View the main server log
docker exec clickhouse cat /var/log/clickhouse-server/clickhouse-server.log

# View error log only
docker exec clickhouse cat /var/log/clickhouse-server/clickhouse-server.err.log
```

## Configuring Log Level

Create `config/logging.xml`:

```xml
<clickhouse>
  <logger>
    <level>information</level>       <!-- trace, debug, information, warning, error -->
    <log>/var/log/clickhouse-server/clickhouse-server.log</log>
    <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
    <size>500M</size>
    <count>10</count>
  </logger>
</clickhouse>
```

Mount it in Docker Compose:

```yaml
volumes:
  - ./config/logging.xml:/etc/clickhouse-server/config.d/logging.xml
  - clickhouse_logs:/var/log/clickhouse-server
```

## Redirecting Logs to Stdout

For Docker log drivers to capture ClickHouse logs, configure logging to stdout:

```xml
<clickhouse>
  <logger>
    <console>1</console>
    <level>information</level>
  </logger>
</clickhouse>
```

## Using Docker Log Drivers

Configure the Docker log driver in docker-compose.yml:

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "5"
```

For forwarding to a centralized system:

```yaml
logging:
  driver: "fluentd"
  options:
    fluentd-address: "localhost:24224"
    tag: "clickhouse"
```

## Querying Logs from Inside ClickHouse

ClickHouse also stores structured query logs in system tables:

```sql
-- Recent errors
SELECT
    event_time,
    type,
    exception,
    query
FROM system.query_log
WHERE type IN ('ExceptionBeforeStart', 'ExceptionWhileProcessing')
  AND event_time > now() - INTERVAL 1 HOUR
ORDER BY event_time DESC
LIMIT 20;
```

## Log Rotation Configuration

The `<size>` and `<count>` settings in the logger config handle log rotation automatically. Size is per-file; count is the number of archived files to keep.

## Summary

Manage ClickHouse Docker logs by configuring log level and rotation in `config.d/logging.xml`, redirecting to stdout for Docker log driver compatibility, and using Docker log driver options to control file size limits. Supplement file logs with `system.query_log` for structured query-level analysis.
