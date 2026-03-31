# How to Ingest Data from Fluentd into ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Fluentd, Log Ingestion, fluent-plugin-clickhouse, Analytics

Description: Learn how to configure Fluentd to forward log data into ClickHouse using the fluent-plugin-clickhouse output plugin with batching and retry.

---

Fluentd is a popular log collector that can forward events to ClickHouse using the `fluent-plugin-clickhouse` output plugin. It supports batching, compression, and automatic retries.

## Installing the Plugin

```bash
gem install fluent-plugin-clickhouse
# or inside a td-agent/fluentd container:
td-agent-gem install fluent-plugin-clickhouse
```

## Creating the Target Table

```sql
CREATE TABLE app_logs (
    ts DateTime,
    level LowCardinality(String),
    service LowCardinality(String),
    message String,
    host String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (service, ts)
TTL ts + INTERVAL 60 DAY;
```

## Configuring Fluentd

```xml
<source>
  @type forward
  port 24224
</source>

<filter app.**>
  @type record_transformer
  <record>
    host "#{Socket.gethostname}"
  </record>
</filter>

<match app.**>
  @type clickhouse
  host clickhouse
  port 8123
  database default
  table app_logs
  username default
  password "#{ENV['CLICKHOUSE_PASSWORD']}"

  flush_interval 5s
  buffer_chunk_limit 8m
  buffer_queue_limit 32
  retry_wait 5s
  retry_limit 5

  <format>
    @type json
  </format>
</match>
```

## Using Fluent Bit Instead

Fluent Bit is lighter and has a native ClickHouse output:

```ini
[OUTPUT]
    Name          clickhouse
    Match         *
    host          clickhouse
    port          8123
    database      default
    table         app_logs
    format        json_stream
    http_user     default
    http_passwd   ${CLICKHOUSE_PASSWORD}
```

## Testing the Pipeline

Send a test event:

```bash
echo '{"ts":"2026-03-31T10:00:00Z","level":"info","service":"api","message":"request received"}' \
  | docker run --rm -i fluent/fluent-bit \
    -i stdin -o clickhouse -p host=clickhouse -p table=app_logs
```

Verify in ClickHouse:

```sql
SELECT service, count() FROM app_logs GROUP BY service;
```

## Summary

Fluentd ingests logs into ClickHouse via `fluent-plugin-clickhouse`. Configure chunk buffering and retries to handle back-pressure. For lightweight deployments, Fluent Bit offers a native ClickHouse output with lower memory overhead.
