# How to Configure Distributed Tracing in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Distributed Tracing, OpenTelemetry, Observability

Description: Configure end-to-end distributed tracing in Rancher to track requests across microservices and identify performance bottlenecks using OpenTelemetry.

## Introduction

Distributed tracing provides visibility into the complete journey of a request across multiple microservices. This guide covers instrumenting applications with OpenTelemetry, configuring trace context propagation, deploying a tracing backend, and creating meaningful dashboards to identify performance bottlenecks in Rancher-managed clusters.

## Prerequisites

- Rancher-managed Kubernetes cluster
- OpenTelemetry Collector deployed
- A tracing backend (Jaeger or Tempo)
- kubectl access

## Step 1: Instrument Applications with OpenTelemetry

### Java Application

```java
// Java app with OpenTelemetry auto-instrumentation
// Add to your Dockerfile or use the OTel Operator
// -javaagent:/opentelemetry-javaagent.jar

// Manual instrumentation example
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.context.Scope;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;

// Configure OpenTelemetry
OtlpGrpcSpanExporter exporter = OtlpGrpcSpanExporter.builder()
    .setEndpoint("http://otel-collector.observability.svc.cluster.local:4317")
    .build();

SdkTracerProvider provider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
    .build();

OpenTelemetry openTelemetry = OpenTelemetrySdk.builder()
    .setTracerProvider(provider)
    .build();

Tracer tracer = openTelemetry.getTracer("order-service");

// Instrument an operation
public OrderResponse processOrder(OrderRequest request) {
    Span span = tracer.spanBuilder("process-order")
        .setAttribute("order.id", request.getOrderId())
        .setAttribute("order.amount", request.getAmount())
        .startSpan();

    try (Scope scope = span.makeCurrent()) {
        // Call inventory service - trace context propagated automatically
        InventoryResponse inventory = inventoryClient.checkInventory(request);

        if (!inventory.isAvailable()) {
            span.setStatus(StatusCode.ERROR, "Inventory not available");
            throw new InsufficientInventoryException();
        }

        span.setAttribute("order.status", "confirmed");
        return confirmOrder(request);
    } catch (Exception e) {
        span.recordException(e);
        span.setStatus(StatusCode.ERROR);
        throw e;
    } finally {
        span.end();
    }
}
```

### Python Application

```python
# Python app with OpenTelemetry
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

# Configure tracer
provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(
            endpoint="http://otel-collector.observability.svc.cluster.local:4317"
        )
    )
)
trace.set_tracer_provider(provider)

# Auto-instrument Flask, HTTP requests, and SQLAlchemy
FlaskInstrumentor().instrument()
RequestsInstrumentor().instrument()
SQLAlchemyInstrumentor().instrument()

# Manual span creation
tracer = trace.get_tracer(__name__)

@app.route('/orders', methods=['POST'])
def create_order():
    with tracer.start_as_current_span("create-order") as span:
        order_data = request.json
        span.set_attribute("order.customer_id", order_data.get('customer_id'))

        # This HTTP call will automatically get trace context injected
        payment_response = requests.post(
            "http://payment-service/charge",
            json={"amount": order_data['amount']}
        )

        if payment_response.status_code != 200:
            span.set_status(trace.Status(trace.StatusCode.ERROR))
            return jsonify({"error": "Payment failed"}), 500

        order_id = save_order(order_data)
        span.set_attribute("order.id", order_id)
        return jsonify({"order_id": order_id})
```

### Node.js Application

```javascript
// Node.js app with OpenTelemetry
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'payment-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0',
    'deployment.environment': 'production',
  }),
  traceExporter: new OTLPTraceExporter({
    url: 'http://otel-collector.observability.svc.cluster.local:4317',
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

## Step 2: Configure Context Propagation

```yaml
# ingress-tracing.yaml - Inject trace context at ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    # Pass trace headers from NGINX
    nginx.ingress.kubernetes.io/configuration-snippet: |
      # Generate trace ID if not present
      set $trace_id $http_x_trace_id;
      if ($trace_id = '') {
        set $trace_id $request_id;
      }
      # Add trace headers to upstream
      proxy_set_header X-Trace-Id $trace_id;
      proxy_set_header X-B3-TraceId $trace_id;
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-gateway
                port:
                  number: 80
```

## Step 3: Configure Sampling Strategies

```yaml
# sampling-deployment.yaml - OTel Collector with tail-based sampling
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-sampler
  namespace: observability
spec:
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317

    processors:
      # Tail-based sampling: make sampling decisions after trace is complete
      tail_sampling:
        decision_wait: 10s
        num_traces: 100000
        policies:
          # Always sample error traces
          - name: errors-policy
            type: status_code
            status_code:
              status_codes: [ERROR]
          # Always sample slow traces (> 2 seconds)
          - name: slow-traces-policy
            type: latency
            latency:
              threshold_ms: 2000
          # Sample 10% of other traces
          - name: probabilistic-policy
            type: probabilistic
            probabilistic:
              sampling_percentage: 10
          # Always sample payment traces
          - name: payment-service-policy
            type: string_attribute
            string_attribute:
              key: service.name
              values:
                - payment-service
              enabled_regex_matching: false
              invert_match: false

    exporters:
      otlp/tempo:
        endpoint: tempo-distributor.observability.svc.cluster.local:4317
        tls:
          insecure: true

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [tail_sampling]
          exporters: [otlp/tempo]
```

## Step 4: Create Grafana Service Map Dashboard

```bash
# Import trace-based service map dashboard
# In Grafana: Explore > Tempo > Service Map

# Create a custom dashboard panel for trace latency
# Dashboard JSON snippet for P99 latency from Tempo-generated metrics
cat > /tmp/trace-dashboard.json << 'EOF'
{
  "targets": [{
    "datasource": {"type": "prometheus"},
    "expr": "histogram_quantile(0.99, sum(rate(traces_spanmetrics_duration_milliseconds_bucket[5m])) by (service, le))",
    "legendFormat": "{{service}} P99"
  }]
}
EOF
```

## Step 5: Monitor Tracing Pipeline Health

```yaml
# tracing-alerts.yaml - Alerts for tracing pipeline
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: tracing-alerts
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  groups:
    - name: distributed-tracing
      rules:
        # Alert if OTel Collector drops spans
        - alert: TracingPipelineDroppingSpans
          expr: |
            sum(rate(otelcol_processor_dropped_spans[5m])) > 100
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "OpenTelemetry Collector is dropping {{ $value }} spans per second"

        # Alert on high trace latency
        - alert: ServiceHighTraceLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(traces_spanmetrics_duration_milliseconds_bucket[5m])) by (service, le)
            ) > 2000
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Service {{ $labels.service }} P99 trace latency is {{ $value }}ms"
```

## Conclusion

Distributed tracing in Rancher provides the missing context needed to debug complex multi-service issues. By instrumenting applications with OpenTelemetry and using tail-based sampling to focus on error and slow traces, you get actionable observability without drowning in trace data. The service map and trace search capabilities in Grafana Tempo enable rapid root cause analysis for latency and error rate spikes across your microservices.
