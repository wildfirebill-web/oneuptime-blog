# How to Trace Redis Operations in Distributed Systems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, OpenTelemetry, Tracing, Distributed System, Observability

Description: Learn how to instrument Redis calls with distributed traces using OpenTelemetry so you can follow requests across services and identify cache bottlenecks.

---

In a distributed system, a single user request may touch multiple services and several Redis calls. Without distributed tracing, it is nearly impossible to know which Redis operation caused a latency spike or contributed to a failed request.

## Why Trace Redis?

- Identify which commands are slow within a larger request trace
- Correlate Redis errors with upstream service failures
- Understand cache hit/miss ratios in context of full request flows

## OpenTelemetry Auto-Instrumentation for Redis (Python)

The `opentelemetry-instrumentation-redis` package automatically wraps Redis calls with spans:

```bash
pip install opentelemetry-sdk opentelemetry-exporter-otlp opentelemetry-instrumentation-redis
```

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.redis import RedisInstrumentor
import redis

# Set up tracing
provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint="http://localhost:4317")))
trace.set_tracer_provider(provider)

# Auto-instrument Redis
RedisInstrumentor().instrument()

r = redis.Redis(host="localhost", port=6379)

tracer = trace.get_tracer("my-service")
with tracer.start_as_current_span("handle-request"):
    r.set("user:42", "alice")
    value = r.get("user:42")
```

Each Redis command becomes a child span with attributes like `db.system=redis`, `db.statement=SET user:42`, and duration.

## Node.js with OpenTelemetry

```bash
npm install @opentelemetry/instrumentation-ioredis @opentelemetry/sdk-node
```

```javascript
const { NodeSDK } = require("@opentelemetry/sdk-node");
const { IORedisInstrumentation } = require("@opentelemetry/instrumentation-ioredis");

const sdk = new NodeSDK({
  instrumentations: [new IORedisInstrumentation()],
});
sdk.start();

const Redis = require("ioredis");
const redis = new Redis();

async function handleRequest() {
  await redis.set("key", "value");
  const val = await redis.get("key");
  return val;
}
```

## Manual Span Creation

When auto-instrumentation is not available, wrap Redis calls manually:

```python
from opentelemetry import trace

tracer = trace.get_tracer("my-service")

def get_user(r, user_id):
    with tracer.start_as_current_span("redis.get_user") as span:
        span.set_attribute("db.system", "redis")
        span.set_attribute("redis.key", f"user:{user_id}")
        result = r.get(f"user:{user_id}")
        span.set_attribute("cache.hit", result is not None)
        return result
```

## Propagating Trace Context Across Services

Ensure your HTTP clients propagate the `traceparent` header so Redis spans appear as children within the correct request tree:

```python
from opentelemetry.propagate import inject
import requests

headers = {}
inject(headers)
requests.get("http://other-service/api/data", headers=headers)
```

## Analyzing Traces

In your trace backend (Jaeger, Tempo, OneUptime), filter traces by `db.system=redis` to find slow commands. Sort by duration to find the worst offenders. A span tree showing a 200ms `HGETALL` inside a 210ms HTTP handler tells you exactly where to optimize.

## Summary

Distributed tracing of Redis operations reveals exactly how cache calls fit into the broader request lifecycle. OpenTelemetry auto-instrumentation is the fastest path to coverage in Python and Node.js. Once spans flow into your tracing backend, you can pinpoint slow Redis commands, track cache hit rates, and correlate Redis errors with service-level incidents.
