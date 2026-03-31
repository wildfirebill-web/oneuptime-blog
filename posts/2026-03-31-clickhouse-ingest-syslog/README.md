# How to Ingest Data from Syslog into ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Syslog, Log Ingestion, Vector, Analytics

Description: Learn how to ingest syslog messages into ClickHouse using Vector or rsyslog with the ClickHouse output plugin for centralized log analytics.

---

Collecting syslog messages into ClickHouse enables fast querying of system logs at scale. The most practical approaches use Vector.dev or rsyslog with a ClickHouse sink.

## Creating the Target Table

```sql
CREATE TABLE syslog (
    received_at DateTime DEFAULT now(),
    host LowCardinality(String),
    facility UInt8,
    severity UInt8,
    app_name String,
    proc_id String,
    msg_id String,
    message String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(received_at)
ORDER BY (host, received_at)
TTL received_at + INTERVAL 90 DAY;
```

## Using Vector.dev

Install Vector and configure a syslog source with ClickHouse sink:

```yaml
# /etc/vector/vector.yaml
sources:
  syslog_in:
    type: syslog
    mode: tcp
    address: "0.0.0.0:514"

sinks:
  clickhouse_out:
    type: clickhouse
    inputs: ["syslog_in"]
    endpoint: "http://clickhouse:8123"
    database: default
    table: syslog
    auth:
      strategy: basic
      user: default
      password: "${CLICKHOUSE_PASSWORD}"
    batch:
      max_events: 5000
      timeout_secs: 5
    encoding:
      timestamp_format: unix
```

Start Vector:

```bash
vector --config /etc/vector/vector.yaml
```

## Using rsyslog with the ClickHouse Output Module

Install the rsyslog ClickHouse module and configure:

```text
# /etc/rsyslog.d/50-clickhouse.conf
module(load="omhttp")

template(name="clickhouseJSON" type="list") {
    constant(value="{")
    constant(value="\"host\":\"") property(name="hostname") constant(value="\",")
    constant(value="\"message\":\"") property(name="msg" format="json") constant(value="\",")
    constant(value="\"severity\":") property(name="syslogseverity")
    constant(value="}")
}

action(
  type="omhttp"
  server="clickhouse"
  serverport="8123"
  restpath="?query=INSERT+INTO+syslog+FORMAT+JSONEachRow"
  template="clickhouseJSON"
  batch.maxsize="500"
)
```

## Verifying Ingestion

```sql
SELECT host, count() AS log_count, max(received_at) AS latest
FROM syslog
GROUP BY host
ORDER BY log_count DESC
LIMIT 10;
```

## Summary

Syslog can be ingested into ClickHouse via Vector (recommended for its native ClickHouse sink and batching) or rsyslog's omhttp module. Define a MergeTree table with TTL to auto-expire old logs, and batch inserts to reduce write amplification.
