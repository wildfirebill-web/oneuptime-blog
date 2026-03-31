# How to Use Structured Logging with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Logging, Structured Log, Observability, JSON

Description: Implement structured logging across Dapr services to produce consistent, queryable log entries that work seamlessly with modern log aggregation platforms.

---

Structured logging replaces unformatted text with machine-readable JSON objects. When combined with Dapr's JSON sidecar logs, structured application logs enable powerful cross-service log queries and correlation using trace IDs.

## Why Structured Logging Matters

With unstructured logs:
```
2026-03-31 10:00:01 Processing order 12345 for customer abc
```

With structured logs:
```json
{"time":"2026-03-31T10:00:01Z","level":"info","msg":"Processing order","order_id":"12345","customer_id":"abc","trace_id":"4bf92f3577b34da6"}
```

The structured format enables queries like `order_id = "12345"` or `customer_id = "abc"` in your log platform.

## Configuring Dapr for JSON Logging

```yaml
metadata:
  annotations:
    dapr.io/log-as-json: "true"
    dapr.io/log-level: "info"
```

## Structured Logging in Go (with zerolog)

```go
package main

import (
    "os"
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

func main() {
    zerolog.TimeFieldFormat = zerolog.TimeFormatUnix
    log.Logger = zerolog.New(os.Stdout).With().
        Str("service", "order-service").
        Timestamp().
        Logger()

    log.Info().
        Str("order_id", "12345").
        Str("customer_id", "abc").
        Float64("amount", 99.99).
        Msg("Processing order")
}
```

Output:
```json
{"level":"info","service":"order-service","order_id":"12345","customer_id":"abc","amount":99.99,"time":1711876801,"message":"Processing order"}
```

## Structured Logging in Python (with structlog)

```python
import structlog

log = structlog.get_logger()
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ]
)

def process_order(order_id: str, customer_id: str):
    log.info("processing_order",
             order_id=order_id,
             customer_id=customer_id,
             service="order-service")
```

## Structured Logging in Node.js (with pino)

```javascript
const pino = require('pino');
const logger = pino({
  level: 'info',
  base: { service: 'order-service' }
});

function processOrder(orderId, customerId) {
  logger.info({ orderId, customerId }, 'Processing order');
}
```

## Including Trace IDs in Application Logs

Extract the Dapr trace ID from request headers and include it in logs:

```go
func handler(w http.ResponseWriter, r *http.Request) {
    traceID := r.Header.Get("traceparent")
    log.Info().
        Str("trace_id", traceID).
        Str("order_id", "12345").
        Msg("Handling request")
}
```

## Defining a Consistent Log Schema

Define a shared log schema across all your services to make cross-service queries predictable:

| Field | Type | Description |
|-------|------|-------------|
| `time` | string | ISO 8601 timestamp |
| `level` | string | debug, info, warn, error |
| `service` | string | Service name matching Dapr app ID |
| `trace_id` | string | W3C trace ID from traceparent |
| `msg` | string | Human-readable message |

Add service-specific fields as needed (e.g., `order_id`, `user_id`).

## Querying Structured Logs

With structured logs in Elasticsearch:

```json
{
  "query": {
    "bool": {
      "must": [
        {"term": {"order_id": "12345"}},
        {"term": {"level": "error"}}
      ]
    }
  }
}
```

In Grafana Loki:

```
{app="order-service"} | json | level="error" | order_id="12345"
```

In Datadog:

```
service:order-service @level:error @order_id:12345
```

## Summary

Structured logging across both Dapr sidecar logs (enabled with `dapr.io/log-as-json`) and application logs creates a consistent JSON schema that enables powerful cross-service queries. Use trace IDs from Dapr's `traceparent` headers to correlate application logs with sidecar logs and distributed traces. Libraries like zerolog, structlog, and pino make it straightforward to produce structured JSON logs in any language.
