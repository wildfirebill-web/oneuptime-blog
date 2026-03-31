# How to Use ClickHouse with Coroot for Infrastructure Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Coroot, Infrastructure Monitoring, Observability, OpenTelemetry

Description: Learn how to configure Coroot to use ClickHouse for storing OpenTelemetry traces and profiles for infrastructure and application performance monitoring.

---

## What Is Coroot

Coroot is an open-source observability tool that automatically maps infrastructure dependencies using eBPF-based telemetry. It uses ClickHouse to store traces, profiles, and log patterns for long-term analysis.

## Architecture

```text
Coroot-Node-Agent (eBPF)  +  OpenTelemetry SDKs
              |
              v
    Coroot Application
              |
              v
    ClickHouse (traces, profiles, logs)
```

## Docker Compose Setup

```yaml
version: '3.8'
services:
  clickhouse:
    image: clickhouse/clickhouse-server:24.3
    environment:
      CLICKHOUSE_USER: coroot
      CLICKHOUSE_PASSWORD: coroot
      CLICKHOUSE_DB: coroot
    ports:
      - "8123:8123"
    volumes:
      - ch-data:/var/lib/clickhouse

  coroot:
    image: ghcr.io/coroot/coroot:latest
    command:
      - --data-dir=/data
      - --bootstrap-clickhouse-address=clickhouse:9000
      - --bootstrap-clickhouse-database=coroot
      - --bootstrap-clickhouse-auth=coroot:coroot
    ports:
      - "8080:8080"
    volumes:
      - coroot-data:/data
    depends_on:
      - clickhouse

  coroot-node-agent:
    image: ghcr.io/coroot/coroot-node-agent:latest
    privileged: true
    pid: host
    network_mode: host
    volumes:
      - /sys/kernel/debug:/sys/kernel/debug:rw
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
      - /proc:/proc:ro
    environment:
      COROOT_ENDPOINT: http://coroot:8080

volumes:
  ch-data:
  coroot-data:
```

## Verifying ClickHouse Tables

```sql
SHOW TABLES FROM coroot;
-- traces
-- profiles
-- log_patterns
-- node_agents
```

## Adjusting Data Retention

```sql
ALTER TABLE coroot.traces
MODIFY TTL toDate(timestamp) + INTERVAL 14 DAY;

ALTER TABLE coroot.profiles
MODIFY TTL toDate(timestamp) + INTERVAL 7 DAY;
```

## Querying Trace Data Directly

```sql
SELECT
  service,
  count() AS span_count,
  countIf(status = 'error') AS errors,
  avg(duration_ms) AS avg_duration
FROM coroot.traces
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY service
ORDER BY errors DESC
```

## Scaling ClickHouse for Coroot

For large Kubernetes clusters with many nodes, increase ClickHouse memory and concurrent query limits:

```xml
<clickhouse>
  <max_concurrent_queries>100</max_concurrent_queries>
  <max_memory_usage>16000000000</max_memory_usage>
</clickhouse>
```

## Summary

Coroot uses ClickHouse to store traces, continuous profiles, and log patterns collected via eBPF and OpenTelemetry. Deploy the `coroot`, `clickhouse`, and `coroot-node-agent` containers together, configure the ClickHouse endpoint at startup, and tune TTL values for storage budget. Direct ClickHouse queries let you build custom dashboards beyond Coroot's built-in views.
