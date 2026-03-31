# How to Ingest Data from Logstash into ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Logstash, Log Ingestion, HTTP Output, Analytics

Description: Learn how to configure Logstash to forward processed log data into ClickHouse using the HTTP output plugin with batching and JSON formatting.

---

Logstash can send processed events to ClickHouse using its HTTP output plugin. While there is no dedicated ClickHouse plugin, the HTTP output targets ClickHouse's native HTTP interface directly.

## Target Table

```sql
CREATE TABLE logs (
    ts DateTime,
    level LowCardinality(String),
    service String,
    host String,
    message String,
    fields String  -- JSON for extra fields
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (service, ts)
TTL ts + INTERVAL 90 DAY;
```

## Logstash Pipeline Configuration

```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
  }
  mutate {
    rename => { "@timestamp" => "ts" }
    add_field => { "fields" => "%{[fields]}" }
  }
}

output {
  http {
    url => "http://clickhouse:8123/?query=INSERT+INTO+logs+FORMAT+JSONEachRow"
    http_method => "post"
    content_type => "application/json"
    format => "json_batch"
    http_compression => true
    batch_size => 1000
    batch_timeout => 5

    headers => {
      "X-ClickHouse-User" => "default"
      "X-ClickHouse-Key" => "${CLICKHOUSE_PASSWORD}"
    }

    codec => json_lines
  }
}
```

## Handling Field Mapping

Logstash events use `@timestamp`. Rename it before sending:

```ruby
filter {
  mutate {
    rename => { "@timestamp" => "ts" }
    convert => { "ts" => "string" }
    gsub => ["ts", "T", " ", "ts", "Z", ""]
  }
}
```

## Testing the Pipeline

```bash
echo '{"message":"test","level":"info","service":"api"}' | \
  logstash -f /etc/logstash/conf.d/clickhouse.conf
```

Then verify in ClickHouse:

```sql
SELECT service, level, count()
FROM logs
WHERE ts >= now() - INTERVAL 10 MINUTE
GROUP BY service, level;
```

## Performance Tips

- Set `batch_size => 5000` and `batch_timeout => 10` for higher throughput
- Enable `async_insert=1` in the ClickHouse URL parameter to reduce part creation
- Use a Buffer table to absorb bursts

## Summary

Logstash forwards events to ClickHouse via the HTTP output plugin targeting the ClickHouse HTTP interface. Use `format => json_batch` and batch settings to reduce request overhead. Rename Logstash's `@timestamp` to match the ClickHouse column name before sending.
