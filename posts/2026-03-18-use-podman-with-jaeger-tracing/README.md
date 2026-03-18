# How to Use Podman with Jaeger for Tracing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Jaeger, Distributed Tracing, Observability, Microservices

Description: Learn how to use Podman with Jaeger to implement distributed tracing for containerized microservices, enabling you to track requests across service boundaries.

---

> Jaeger running in Podman containers brings distributed tracing to your microservices, letting you visualize request flows, identify bottlenecks, and debug latency issues across service boundaries.

When your application consists of multiple containerized microservices, understanding how a single request flows through the system becomes challenging. Distributed tracing solves this by assigning a unique trace ID to each request and recording timing information as it passes through each service. Jaeger is one of the most popular open-source distributed tracing platforms, and running it in Podman containers makes it easy to deploy alongside your application services.

---

## Deploying Jaeger All-in-One

For development and small deployments, Jaeger provides an all-in-one image that includes the collector, query service, and UI:

```bash
podman run -d \
  --name jaeger \
  --restart always \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  -p 14250:14250 \
  -p 14268:14268 \
  -p 14269:14269 \
  jaegertracing/all-in-one:latest
```

Key ports:
- `6831/udp` - Jaeger agent compact thrift
- `4317` - OpenTelemetry gRPC
- `4318` - OpenTelemetry HTTP
- `16686` - Jaeger UI
- `14268` - Jaeger collector HTTP

Access the Jaeger UI at `http://localhost:16686`.

## Instrumenting a Python Application

Add tracing to a Python Flask application:

```bash
pip install opentelemetry-api opentelemetry-sdk \
    opentelemetry-exporter-otlp-proto-grpc \
    opentelemetry-instrumentation-flask \
    opentelemetry-instrumentation-requests
```

```python
# app.py
from flask import Flask, request, jsonify
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.sdk.resources import Resource
import requests as req_lib
import time

# Configure tracing
resource = Resource.create({"service.name": "api-gateway"})
provider = TracerProvider(resource=resource)
exporter = OTLPSpanExporter(endpoint="http://jaeger:4317", insecure=True)
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

@app.route('/api/orders', methods=['POST'])
def create_order():
    with tracer.start_as_current_span("validate-order") as span:
        order_data = request.json
        span.set_attribute("order.items_count", len(order_data.get("items", [])))

    with tracer.start_as_current_span("check-inventory"):
        inventory_response = req_lib.get("http://inventory-service:3001/check",
            params={"items": ",".join(order_data.get("items", []))})

    with tracer.start_as_current_span("process-payment"):
        payment_response = req_lib.post("http://payment-service:3002/charge",
            json={"amount": order_data.get("total", 0)})

    with tracer.start_as_current_span("save-order") as span:
        time.sleep(0.05)  # Simulate database write
        order_id = "ORD-12345"
        span.set_attribute("order.id", order_id)

    return jsonify({"order_id": order_id, "status": "created"})

@app.route('/health')
def health():
    return jsonify({"status": "healthy"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)
```

## Instrumenting a Node.js Service

```javascript
// tracing.js
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { BatchSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { Resource } = require('@opentelemetry/resources');
const { ATTR_SERVICE_NAME } = require('@opentelemetry/semantic-conventions');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');

const provider = new NodeTracerProvider({
    resource: new Resource({
        [ATTR_SERVICE_NAME]: 'inventory-service',
    }),
});

const exporter = new OTLPTraceExporter({
    url: 'http://jaeger:4317',
});

provider.addSpanProcessor(new BatchSpanProcessor(exporter));
provider.register();

registerInstrumentations({
    instrumentations: [
        new HttpInstrumentation(),
        new ExpressInstrumentation(),
    ],
});

module.exports = { provider };
```

```javascript
// server.js
require('./tracing');
const express = require('express');
const { trace } = require('@opentelemetry/api');

const app = express();
const tracer = trace.getTracer('inventory-service');

app.get('/check', (req, res) => {
    const span = tracer.startSpan('check-stock-levels');

    const items = (req.query.items || '').split(',');
    span.setAttribute('items.count', items.length);

    // Simulate inventory check
    const result = items.map(item => ({
        item,
        available: Math.random() > 0.1,
        quantity: Math.floor(Math.random() * 100),
    }));

    span.end();
    res.json({ inventory: result });
});

app.listen(3001, () => console.log('Inventory service on port 3001'));
```

## Multi-Service Deployment with Tracing

Deploy the complete traced application stack:

```yaml
# tracing-stack.yml
version: "3"
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    restart: always
    ports:
      - "16686:16686"
      - "4317:4317"
      - "4318:4318"

  api-gateway:
    build:
      context: ./api-gateway
    restart: always
    ports:
      - "3000:3000"
    environment:
      OTEL_EXPORTER_OTLP_ENDPOINT: http://jaeger:4317
      OTEL_SERVICE_NAME: api-gateway
    depends_on:
      - jaeger
      - inventory-service
      - payment-service

  inventory-service:
    build:
      context: ./inventory-service
    restart: always
    environment:
      OTEL_EXPORTER_OTLP_ENDPOINT: http://jaeger:4317
      OTEL_SERVICE_NAME: inventory-service
    depends_on:
      - jaeger

  payment-service:
    build:
      context: ./payment-service
    restart: always
    environment:
      OTEL_EXPORTER_OTLP_ENDPOINT: http://jaeger:4317
      OTEL_SERVICE_NAME: payment-service
    depends_on:
      - jaeger

  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: dbpass
```

## Production Jaeger Deployment

For production, use separate components with persistent storage:

```yaml
# jaeger-production.yml
version: "3"
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    restart: always
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    volumes:
      - es-data:/usr/share/elasticsearch/data

  jaeger-collector:
    image: jaegertracing/jaeger-collector:latest
    restart: always
    ports:
      - "14268:14268"
      - "4317:4317"
      - "4318:4318"
    environment:
      SPAN_STORAGE_TYPE: elasticsearch
      ES_SERVER_URLS: http://elasticsearch:9200
    depends_on:
      - elasticsearch

  jaeger-query:
    image: jaegertracing/jaeger-query:latest
    restart: always
    ports:
      - "16686:16686"
    environment:
      SPAN_STORAGE_TYPE: elasticsearch
      ES_SERVER_URLS: http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  es-data:
```

## Querying Traces via API

Use the Jaeger API to query traces programmatically:

```bash
# Get all services
curl -s http://localhost:16686/api/services | jq .

# Get traces for a service
curl -s "http://localhost:16686/api/traces?service=api-gateway&limit=20" | jq .

# Get a specific trace
curl -s "http://localhost:16686/api/traces/abc123def456" | jq .

# Search traces with parameters
curl -s "http://localhost:16686/api/traces?service=api-gateway&operation=POST%20/api/orders&minDuration=100ms&limit=10" | jq .
```

## Sampling Strategies

Configure trace sampling to manage volume in production:

```bash
# Create a sampling strategies file
cat > /tmp/sampling-strategies.json << 'EOF'
{
  "default_strategy": {
    "type": "probabilistic",
    "param": 0.1
  }
}
EOF

podman run -d \
  --name jaeger \
  -v /tmp/sampling-strategies.json:/etc/jaeger/sampling-strategies.json:ro,Z \
  -e SAMPLING_STRATEGIES_FILE=/etc/jaeger/sampling-strategies.json \
  jaegertracing/all-in-one:latest
```

For more control, use a per-service sampling configuration:

```json
{
  "service_strategies": [
    {
      "service": "api-gateway",
      "type": "probabilistic",
      "param": 0.5
    },
    {
      "service": "payment-service",
      "type": "probabilistic",
      "param": 1.0
    }
  ],
  "default_strategy": {
    "type": "probabilistic",
    "param": 0.1
  }
}
```

## Conclusion

Jaeger with Podman makes distributed tracing accessible for containerized microservice architectures. The all-in-one deployment is perfect for development, while separate collector, query, and storage components scale for production use. With OpenTelemetry instrumentation libraries available for every major language, adding tracing to your services requires minimal code changes. The visibility that distributed tracing provides into request flows, latency bottlenecks, and error propagation makes it an essential tool for operating microservices reliably.
