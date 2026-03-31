# How to Build a Log Search System with RediSearch

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Log, Search

Description: Build a real-time log ingestion and search system using RediSearch with full-text search, severity filtering, and time-range queries.

---

Logs are most valuable when they are searchable in real time. RediSearch can index logs stored as Redis hashes, providing full-text search, severity filtering, and time-range queries with millisecond latency.

## Creating the Log Index

```bash
FT.CREATE log_idx ON HASH PREFIX 1 log:
  SCHEMA
    message TEXT
    service TAG
    level TAG
    host TAG
    timestamp NUMERIC SORTABLE
    trace_id TAG
```

## Ingesting Logs

Write a log ingester that stores each log event as a hash:

```python
import redis
import time
import uuid

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Keep a counter for unique log IDs
LOG_COUNTER_KEY = "log:counter"

def ingest_log(message: str, service: str, level: str,
               host: str, trace_id: str = None):
    log_id = r.incr(LOG_COUNTER_KEY)
    r.hset(f"log:{log_id}", mapping={
        "message": message,
        "service": service,
        "level": level,
        "host": host,
        "timestamp": int(time.time() * 1000),
        "trace_id": trace_id or str(uuid.uuid4())
    })
    return log_id

# Ingest some sample logs
ingest_log("Connection refused to database", "api", "ERROR",
           "app-01", "abc-123")
ingest_log("Request processed in 45ms", "api", "INFO",
           "app-01", "abc-456")
ingest_log("Disk usage above 90%", "monitor", "WARN",
           "worker-02")
```

## Full-Text Search

Search across all log messages:

```python
def search_logs(query: str, limit: int = 50):
    result = r.execute_command(
        'FT.SEARCH', 'log_idx', query,
        'SORTBY', 'timestamp', 'DESC',
        'LIMIT', 0, limit
    )
    count = result[0]
    logs = []
    for i in range(1, len(result), 2):
        fields = dict(zip(result[i+1][0::2], result[i+1][1::2]))
        logs.append(fields)
    return {"count": count, "logs": logs}

errors = search_logs("connection refused")
```

## Filtering by Severity and Service

```python
def get_errors_for_service(service: str, level: str = "ERROR",
                            limit: int = 100):
    query = f"@service:{{{service}}} @level:{{{level}}}"
    return r.execute_command(
        'FT.SEARCH', 'log_idx', query,
        'SORTBY', 'timestamp', 'DESC',
        'LIMIT', 0, limit
    )

api_errors = get_errors_for_service("api", "ERROR")
```

## Time-Range Queries

Retrieve logs within a specific time window:

```python
def get_logs_in_range(start_ms: int, end_ms: int,
                      service: str = None, level: str = None):
    parts = [f"@timestamp:[{start_ms} {end_ms}]"]
    if service:
        parts.append(f"@service:{{{service}}}")
    if level:
        parts.append(f"@level:{{{level}}}")

    query = " ".join(parts)
    return r.execute_command(
        'FT.SEARCH', 'log_idx', query,
        'SORTBY', 'timestamp', 'ASC',
        'LIMIT', 0, 500
    )

# Last 5 minutes
now_ms = int(time.time() * 1000)
recent = get_logs_in_range(now_ms - 300_000, now_ms)
```

## Searching by Trace ID

Correlate all logs for a specific request trace:

```python
def get_logs_by_trace(trace_id: str):
    return r.execute_command(
        'FT.SEARCH', 'log_idx', f"@trace_id:{{{trace_id}}}",
        'SORTBY', 'timestamp', 'ASC',
        'LIMIT', 0, 100
    )
```

## Managing Log Retention

Set TTL on log hashes to enforce retention policies:

```python
def ingest_log_with_ttl(message: str, service: str, level: str,
                        host: str, ttl_seconds: int = 604800):
    log_id = ingest_log(message, service, level, host)
    r.expire(f"log:{log_id}", ttl_seconds)
    return log_id
```

## Summary

RediSearch turns Redis into a capable log search platform suitable for development environments and moderate production workloads. By combining full-text search with TAG and NUMERIC filters, you can build expressive log queries without leaving Redis. Use TTL on log hashes to automatically expire old entries and control memory usage.
