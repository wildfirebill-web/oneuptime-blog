# How to Correlate Logs with Traces in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Logging, Tracing, Observability, OpenTelemetry

Description: Link Dapr service logs to distributed traces using trace IDs to enable seamless navigation from logs to traces during incident investigation.

---

Log-trace correlation lets you jump from a log entry showing an error directly to the distributed trace for that request. Dapr propagates W3C TraceContext headers, which carry the trace ID that you can embed in application logs to achieve full correlation.

## How Dapr Provides Trace IDs

When Dapr handles a service invocation or pub/sub message, it propagates the `traceparent` header using the W3C TraceContext format:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^  ^^
             version  trace-id (32 hex chars)        span-id          flags
```

The trace ID is the 32-character hex string in the middle.

## Extracting the Trace ID in Go

```go
package main

import (
    "net/http"
    "go.opentelemetry.io/otel/trace"
    "github.com/rs/zerolog/log"
)

func orderHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    span := trace.SpanFromContext(ctx)
    traceID := span.SpanContext().TraceID().String()
    spanID := span.SpanContext().SpanID().String()

    log.Info().
        Str("trace_id", traceID).
        Str("span_id", spanID).
        Str("order_id", "12345").
        Msg("Processing order")
}
```

## Extracting the Trace ID in Python

```python
from opentelemetry import trace
import structlog

log = structlog.get_logger()

def process_order(order_id: str):
    span = trace.get_current_span()
    ctx = span.get_span_context()

    log.info("processing_order",
             trace_id=format(ctx.trace_id, '032x'),
             span_id=format(ctx.span_id, '016x'),
             order_id=order_id)
```

## Extracting the Trace ID in Node.js

```javascript
const { trace } = require('@opentelemetry/api');
const pino = require('pino');
const logger = pino();

function processOrder(orderId) {
  const span = trace.getActiveSpan();
  const ctx = span?.spanContext();

  logger.info({
    traceId: ctx?.traceId,
    spanId: ctx?.spanId,
    orderId
  }, 'Processing order');
}
```

## Correlating in Grafana (Logs + Traces)

Configure a Loki data source with a derived field to link trace IDs to Tempo:

```yaml
# Grafana data source config for Loki
datasources:
  - name: Loki
    type: loki
    url: http://loki:3100
    jsonData:
      derivedFields:
        - datasourceUid: tempo
          matcherRegex: '"trace_id":"([a-f0-9]{32})"'
          name: TraceID
          url: "$${__value.raw}"
```

With this config, clicking a trace ID in a Loki log line opens the trace in Tempo.

## Correlating in Elasticsearch/Kibana

Query logs by trace ID in Kibana:

```json
{
  "query": {
    "term": {
      "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"
    }
  }
}
```

Create a Kibana Canvas or Dashboard that links the `trace_id` field to your Jaeger or Zipkin instance:

```
https://jaeger.yourcompany.com/trace/${trace_id}
```

## Dapr Sidecar Log Correlation

Dapr sidecar logs also include trace IDs when a request is being processed. Look for the `traceId` field:

```bash
kubectl logs deploy/order-service -c daprd | \
  jq -c 'select(.traceId == "4bf92f3577b34da6a3ce929d0e0e4736")'
```

## Summary

Log-trace correlation in Dapr services requires extracting the trace ID from the OpenTelemetry span context and embedding it in structured application logs. Both sidecar logs and application logs will then share the same trace ID, enabling navigation from a log error to the full distributed trace. Configure your log visualization tool (Grafana, Kibana, Datadog) to recognize trace IDs as clickable links to your tracing backend.
