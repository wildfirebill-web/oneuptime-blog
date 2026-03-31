# How to Correlate Redis Operations with Application Logs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Logging, Observability, OpenTelemetry, Correlation

Description: Learn how to correlate Redis command logs with application logs using trace IDs and structured logging so you can debug issues across your entire stack.

---

When a Redis error occurs, you need to know which application request triggered it. Correlating Redis operation logs with application logs using a shared trace or request ID makes this possible.

## The Problem

Without correlation, your logs look like:

```text
[app] ERROR: Failed to process order 99
[redis] WRONGTYPE Operation against a key holding the wrong kind of value
```

These two lines are related, but there is no way to know that from the logs alone.

## Step 1: Add Request IDs to Every Log Entry

Use middleware to generate a request ID and attach it to all logs for the request lifecycle:

```python
import uuid
import logging
from contextvars import ContextVar

request_id_var: ContextVar[str] = ContextVar("request_id", default="")

class RequestIDFilter(logging.Filter):
    def filter(self, record):
        record.request_id = request_id_var.get("")
        return True

logging.basicConfig(format="%(asctime)s [%(request_id)s] %(levelname)s %(message)s")
logging.getLogger().addFilter(RequestIDFilter())
```

In your request handler:

```python
def handle_request(request):
    request_id_var.set(str(uuid.uuid4()))
    # now all logs from this request include the request_id
```

## Step 2: Include Request ID in Redis Operation Logs

Wrap your Redis client to log commands with the current request ID:

```python
import redis
import logging
import json
import time

logger = logging.getLogger("redis.ops")

class CorrelatedRedis(redis.Redis):
    def execute_command(self, *args, **options):
        start = time.monotonic()
        try:
            result = super().execute_command(*args, **options)
            status = "ok"
        except Exception as e:
            status = f"error: {e}"
            raise
        finally:
            logger.info(json.dumps({
                "request_id": request_id_var.get(""),
                "command": args[0],
                "key": args[1] if len(args) > 1 else None,
                "status": status,
                "duration_ms": round((time.monotonic() - start) * 1000, 2)
            }))
        return result
```

## Step 3: Use OpenTelemetry Trace IDs for Correlation

If you use OpenTelemetry, trace IDs are the industry-standard correlation handle:

```python
from opentelemetry import trace as otel_trace

class OTelCorrelatedRedis(redis.Redis):
    def execute_command(self, *args, **options):
        span = otel_trace.get_current_span()
        ctx = span.get_span_context()
        trace_id = format(ctx.trace_id, "032x") if ctx.is_valid else ""
        logger.info(json.dumps({
            "trace_id": trace_id,
            "command": args[0],
            "key": args[1] if len(args) > 1 else None,
        }))
        return super().execute_command(*args, **options)
```

Now your Redis logs carry the same `trace_id` as your application logs, letting you query both in Loki or Elasticsearch with a single filter.

## Step 4: Querying Correlated Logs

In Grafana Loki, find all logs for a trace:

```text
{app="myapp"} | json | trace_id="4bf92f3577b34da6a3ce929d0e0e4736"
```

In Elasticsearch:

```text
GET /logs/_search
{
  "query": {
    "term": { "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736" }
  }
}
```

This returns both application and Redis logs for that single request, making root cause analysis straightforward.

## Summary

Correlating Redis operations with application logs requires a shared identifier - either a request ID or an OpenTelemetry trace ID - injected into every log line. Wrapping your Redis client to emit structured logs with this identifier enables cross-stack debugging. Combined with a log aggregation tool, you can find the exact Redis command that caused any application error.
