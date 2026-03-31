# How to Implement Distributed Tracing Context with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Distributed Tracing, Observability, Microservice, OpenTelemetry

Description: Use Redis to propagate and store distributed trace context across microservices when direct HTTP header propagation is not possible, such as in async queues.

---

In synchronous HTTP calls, trace context travels in headers (W3C `traceparent`). But in async workflows - message queues, batch jobs, or fire-and-forget tasks - headers are not available. Redis provides a lightweight store for trace context that any downstream service can retrieve.

## The Problem: Async Context Loss

```text
Service A  -->  Redis Queue (no headers)  -->  Worker B
   |                                              |
trace-id=abc                               no trace-id
```

Worker B cannot link its spans back to Service A's trace without context propagation.

## Storing Trace Context Alongside Work Items

When enqueuing work, store the trace context in Redis alongside the job:

```python
from opentelemetry import trace
from opentelemetry.propagate import inject
import redis
import json

r = redis.Redis(host="redis", port=6379, decode_responses=True)
tracer = trace.get_tracer(__name__)

def enqueue_job(job_id: str, payload: dict):
    with tracer.start_as_current_span("enqueue_job") as span:
        # Collect trace context headers
        carrier = {}
        inject(carrier)  # adds traceparent, tracestate

        # Store context alongside the job
        r.hset(f"job:{job_id}", mapping={
            "payload": json.dumps(payload),
            "trace_context": json.dumps(carrier)
        })
        r.rpush("job_queue", job_id)
```

## Restoring Context in the Worker

```python
from opentelemetry.propagate import extract

def process_job(job_id: str):
    job_data = r.hgetall(f"job:{job_id}")
    carrier = json.loads(job_data["trace_context"])

    # Restore trace context from Redis
    ctx = extract(carrier)  # reconstructs the parent span context

    with tracer.start_as_current_span(
        "process_job",
        context=ctx,
        kind=trace.SpanKind.CONSUMER
    ) as span:
        span.set_attribute("job.id", job_id)
        payload = json.loads(job_data["payload"])
        # process...
```

## Storing Active Trace IDs for Correlation

Sometimes you want to look up the trace ID for an active operation from any service:

```python
def register_active_trace(correlation_id: str, trace_id: str, ttl: int = 300):
    r.setex(f"trace:{correlation_id}", ttl, trace_id)

def get_trace_id(correlation_id: str) -> str | None:
    return r.get(f"trace:{correlation_id}")
```

```bash
# Look up trace by order ID for debugging
redis-cli GET trace:order-8421
# "4bf92f3577b34da6a3ce929d0e0e4736"
```

## Fan-Out Trace Linking

When one event spawns multiple workers:

```python
def fan_out_jobs(parent_trace_carrier: dict, job_ids: list):
    pipeline = r.pipeline()
    for job_id in job_ids:
        pipeline.hset(f"job:{job_id}", "trace_context",
                      json.dumps(parent_trace_carrier))
    pipeline.execute()
```

All worker spans link back to the same parent span, keeping the distributed trace intact.

## TTL Management

Trace context in Redis should have a TTL slightly longer than the expected processing time:

```python
r.expire(f"job:{job_id}", 3600)  # 1 hour for long-running jobs
```

After a job completes, delete the key to avoid unbounded growth:

```python
r.delete(f"job:{job_id}")
```

## Summary

Redis bridges the trace context gap in async microservice workflows by storing W3C `traceparent` headers alongside queued work items. Workers extract the context on pickup and start child spans linked to the originating trace. This gives you end-to-end visibility in your observability platform even when work crosses asynchronous boundaries like queues or batch processors.
