# How to Set Up Distributed Tracing with Jaeger via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Jaeger, Distributed Tracing, Observability, Microservices

Description: Deploy Jaeger for distributed tracing via Portainer to visualize request flows across microservices, identify latency bottlenecks, and debug production issues.

## Introduction

Distributed tracing follows a request as it traverses multiple microservices, recording timing and context at each hop. When a user reports a slow request, distributed tracing shows exactly which service caused the delay and why. Jaeger is the most widely deployed open-source distributed tracing system. This guide covers deploying Jaeger via Portainer and instrumenting services to send traces.

## Step 1: Deploy Jaeger All-In-One (Development)

```yaml
# docker-compose.yml - Jaeger all-in-one for development

version: "3.8"

services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    restart: unless-stopped
    environment:
      - COLLECTOR_OTLP_ENABLED=true
      - COLLECTOR_ZIPKIN_HOST_PORT=:9411
      # In-memory storage (dev only - data lost on restart)
      - SPAN_STORAGE_TYPE=memory
      - MEMORY_MAX_TRACES=50000
    ports:
      - "16686:16686"   # Jaeger UI
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
      - "14268:14268"   # Jaeger HTTP collector
      - "9411:9411"     # Zipkin compatible
    networks:
      - tracing_net

networks:
  tracing_net:
    driver: bridge
    name: tracing_net
```

## Step 2: Deploy Jaeger with Elasticsearch Storage (Production)

```yaml
# docker-compose.yml - Production Jaeger with Elasticsearch
version: "3.8"

services:
  # Jaeger Collector: receives spans from applications
  jaeger-collector:
    image: jaegertracing/jaeger-collector:latest
    container_name: jaeger_collector
    restart: unless-stopped
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
      - ES_INDEX_PREFIX=jaeger
      - ES_NUM_SHARDS=1
      - ES_NUM_REPLICAS=0
      - COLLECTOR_OTLP_ENABLED=true
    ports:
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
      - "14268:14268"   # Jaeger HTTP
    networks:
      - tracing_net
    depends_on:
      elasticsearch:
        condition: service_healthy

  # Jaeger Query: serves the UI and API
  jaeger-query:
    image: jaegertracing/jaeger-query:latest
    container_name: jaeger_query
    restart: unless-stopped
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
      - ES_INDEX_PREFIX=jaeger
    ports:
      - "16686:16686"   # Jaeger UI
      - "16685:16685"   # Jaeger gRPC API
    networks:
      - tracing_net
    depends_on:
      - jaeger-collector

  # Elasticsearch storage backend
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: jaeger_elasticsearch
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    volumes:
      - jaeger_es_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - tracing_net
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200/_cluster/health | grep -q green"]
      interval: 30s
      timeout: 10s
      retries: 10

volumes:
  jaeger_es_data:

networks:
  tracing_net:
    driver: bridge
    name: tracing_net
```

## Step 3: Instrument a Python FastAPI Service

```python
# main.py - FastAPI with OpenTelemetry tracing to Jaeger
from fastapi import FastAPI
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
import httpx

# Configure tracing
resource = Resource(attributes={
    "service.name": "user-service",
    "service.version": "1.0.0",
    "deployment.environment": "production"
})

provider = TracerProvider(resource=resource)
provider.add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(
            endpoint="http://jaeger-collector:4317",
            insecure=True
        )
    )
)
trace.set_tracer_provider(provider)

app = FastAPI()
# Auto-instrument FastAPI (traces all requests)
FastAPIInstrumentor.instrument_app(app)
# Auto-instrument outgoing HTTP calls
HTTPXClientInstrumentor().instrument()

tracer = trace.get_tracer(__name__)

@app.get("/users/{user_id}")
async def get_user(user_id: str):
    # Custom span for business logic
    with tracer.start_as_current_span("fetch_user_from_db") as span:
        span.set_attribute("user.id", user_id)
        span.set_attribute("db.type", "postgresql")

        # Simulate database call
        user = {"id": user_id, "name": "Alice"}
        span.set_attribute("db.rows_returned", 1)

        return user

@app.get("/users/{user_id}/orders")
async def get_user_orders(user_id: str):
    with tracer.start_as_current_span("get_user_orders") as span:
        span.set_attribute("user.id", user_id)

        # Call another service - trace propagates automatically
        async with httpx.AsyncClient() as client:
            response = await client.get(f"http://order-service/orders?user={user_id}")

        orders = response.json()
        span.set_attribute("orders.count", len(orders))
        return orders
```

## Step 4: Instrument a Node.js Service

```javascript
// tracing.js - Initialize before other imports
const { NodeSDK } = require("@opentelemetry/sdk-node");
const {
  OTLPTraceExporter,
} = require("@opentelemetry/exporter-trace-otlp-grpc");
const {
  getNodeAutoInstrumentations,
} = require("@opentelemetry/auto-instrumentations-node");
const { Resource } = require("@opentelemetry/resources");
const {
  SemanticResourceAttributes,
} = require("@opentelemetry/semantic-conventions");

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: "order-service",
    [SemanticResourceAttributes.SERVICE_VERSION]: "2.0.0",
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: "production",
  }),
  traceExporter: new OTLPTraceExporter({
    url: "http://jaeger-collector:4317",
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      "@opentelemetry/instrumentation-express": { enabled: true },
      "@opentelemetry/instrumentation-http": { enabled: true },
      "@opentelemetry/instrumentation-pg": { enabled: true },
    }),
  ],
});

sdk.start();
process.on("SIGTERM", () => sdk.shutdown());
```

## Step 5: Deploy Instrumented Services

```yaml
# docker-compose.yml - Instrumented microservices
version: "3.8"

services:
  user-service:
    image: myapp/user-service:latest
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger-collector:4317
      - OTEL_SERVICE_NAME=user-service
      - OTEL_TRACES_SAMPLER=parentbased_traceidratio
      - OTEL_TRACES_SAMPLER_ARG=0.1    # Sample 10% in production
    networks:
      - app_net
      - tracing_net

  order-service:
    image: myapp/order-service:latest
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger-collector:4317
      - OTEL_SERVICE_NAME=order-service
      - OTEL_TRACES_SAMPLER=parentbased_traceidratio
      - OTEL_TRACES_SAMPLER_ARG=0.1
    networks:
      - app_net
      - tracing_net

networks:
  app_net:
    driver: bridge
  tracing_net:
    external: true
```

## Step 6: Use Jaeger UI to Investigate Traces

```bash
# Access Jaeger UI
open http://localhost:16686

# API queries to find slow traces
curl -s "http://localhost:16686/api/traces?service=user-service&minDuration=1s" | \
  jq '.data[] | {traceID: .traceID, duration: .spans[0].duration}'

# Find traces with errors
curl -s "http://localhost:16686/api/traces?service=user-service&tags=%7B%22error%22%3A%22true%22%7D" | \
  jq '.data[].traceID'

# Get a specific trace
curl -s "http://localhost:16686/api/traces/YOUR_TRACE_ID" | \
  jq '.data[].spans[] | {operationName, duration, tags}'
```

## Conclusion

Jaeger transforms debugging distributed systems from "which service caused this?" to "I can see exactly where the 2-second delay happened and why." The combination of automatic instrumentation (via OpenTelemetry SDK) for HTTP, database, and message queue calls with custom spans for business logic provides complete trace coverage. Deploy Jaeger all-in-one for development and the split collector/query architecture with Elasticsearch for production. Portainer manages both the tracing infrastructure and the instrumented application services from a single dashboard, making it easy to roll out new service versions with updated instrumentation.
