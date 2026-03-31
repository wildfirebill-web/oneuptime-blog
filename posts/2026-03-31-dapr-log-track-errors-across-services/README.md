# How to Log and Track Errors Across Dapr Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Observability, Logging, Tracing, Error Tracking

Description: Learn how to centralize error logging and tracking across Dapr microservices using distributed tracing, structured logs, and correlation IDs.

---

## Observability Challenges in Dapr

In a Dapr-based microservices architecture, a single user request can span five or more services. When something goes wrong, you need to correlate logs and errors across all of them. Dapr automatically propagates W3C trace context headers, giving you a correlation ID that ties together logs from every service involved in a request.

## Structured Error Logging with Trace Context

Extract the trace context from Dapr-forwarded requests and include it in every log line:

```python
import logging
import json
from flask import Flask, request

app = Flask(__name__)
logger = logging.getLogger("order-service")

def get_trace_context() -> dict:
    return {
        "traceId": request.headers.get("traceparent", "unknown"),
        "daprAppId": request.headers.get("dapr-app-id", "unknown")
    }

@app.route("/process-order", methods=["POST"])
def process_order():
    trace = get_trace_context()
    order = request.get_json()

    logger.info(json.dumps({
        "event": "order.processing.started",
        "orderId": order.get("orderId"),
        **trace
    }))

    try:
        result = fulfill_order(order)
        logger.info(json.dumps({
            "event": "order.processing.completed",
            "orderId": order.get("orderId"),
            **trace
        }))
        return result
    except Exception as e:
        logger.error(json.dumps({
            "event": "order.processing.failed",
            "orderId": order.get("orderId"),
            "error": str(e),
            "errorType": type(e).__name__,
            **trace
        }))
        raise
```

## Configuring Dapr Tracing

Enable Zipkin-compatible tracing in your Dapr configuration:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: tracing-config
  namespace: orders
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      endpointAddress: "http://jaeger-collector:9411/api/v2/spans"
```

## Centralized Error Aggregation with Pub/Sub

Each service publishes error events to a central aggregation topic:

```python
from dapr.clients import DaprClient
import traceback
import json

def report_error(service_id: str, error: Exception, context: dict):
    with DaprClient() as client:
        client.publish_event(
            pubsub_name="platform-pubsub",
            topic_name="service-errors",
            data=json.dumps({
                "serviceId": service_id,
                "errorType": type(error).__name__,
                "message": str(error),
                "stackTrace": traceback.format_exc(),
                "context": context,
                "traceId": context.get("traceId")
            }),
            data_content_type="application/json"
        )
```

## Error Dashboard with Dapr State Store

An aggregation service tracks error rates per service:

```python
@app.route("/service-errors", methods=["POST"])
def aggregate_error():
    event = request.get_json().get("data", {})
    service_id = event.get("serviceId")

    with DaprClient() as client:
        key = f"error-stats:{service_id}"
        raw = client.get_state(store_name="metrics-state", key=key)
        stats = json.loads(raw.data) if raw.data else {"total": 0, "byType": {}}

        stats["total"] += 1
        error_type = event.get("errorType", "Unknown")
        stats["byType"][error_type] = stats["byType"].get(error_type, 0) + 1

        client.save_state(store_name="metrics-state", key=key, value=json.dumps(stats))

    return "", 200
```

## Querying Error Logs with Labels

Use structured log queries in Loki or CloudWatch Insights:

```bash
# Loki query - all errors for a specific trace
{namespace="orders"} | json | event="order.processing.failed" | traceId="00-abc123-01"

# CloudWatch Insights
fields @timestamp, event, orderId, error, traceId
| filter event like /failed/
| sort @timestamp desc
| limit 50
```

## Summary

Dapr's automatic trace context propagation provides the correlation IDs needed to track errors across services. Combining structured JSON logging with trace headers and a central error aggregation topic gives you a complete picture of failures from the initial request through every downstream call. State-backed error counters feed real-time dashboards without requiring a dedicated metrics pipeline.
