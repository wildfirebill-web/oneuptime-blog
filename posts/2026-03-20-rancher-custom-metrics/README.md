# How to Set Up Custom Metrics Collection in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Custom Metrics, Prometheus, Monitoring, Observability

Description: Configure custom application metrics collection in Rancher using Prometheus ServiceMonitors, PodMonitors, and custom exporters for business-specific observability.

## Introduction

Beyond infrastructure metrics, custom application metrics provide business-level visibility: orders processed, payment failures, active sessions, and API response times specific to your application. This guide covers implementing custom metrics in your applications, configuring Prometheus to scrape them, and creating Grafana dashboards in Rancher.

## Prerequisites

- Rancher Monitoring (Prometheus/Grafana) installed
- kubectl access
- Applications you want to instrument

## Step 1: Expose Metrics in Your Application

### Python (Flask) Application

```python
# app.py - Flask app with Prometheus metrics
from flask import Flask, request
from prometheus_client import Counter, Histogram, Gauge, make_wsgi_app
from werkzeug.middleware.dispatcher import DispatcherMiddleware
import time

app = Flask(__name__)

# Define custom metrics
ORDER_COUNTER = Counter(
    'orders_processed_total',
    'Total orders processed',
    ['status', 'product_category']
)

ORDER_LATENCY = Histogram(
    'order_processing_seconds',
    'Time spent processing orders',
    ['product_category'],
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 2.0, 5.0, 10.0]
)

ACTIVE_SESSIONS = Gauge(
    'active_user_sessions',
    'Current number of active user sessions'
)

PAYMENT_FAILURES = Counter(
    'payment_failures_total',
    'Total payment processing failures',
    ['reason', 'gateway']
)

@app.route('/orders', methods=['POST'])
def create_order():
    start_time = time.time()
    category = request.json.get('category', 'unknown')

    try:
        order = process_order(request.json)

        # Record successful order
        ORDER_COUNTER.labels(status='success', product_category=category).inc()
        ORDER_LATENCY.labels(product_category=category).observe(time.time() - start_time)

        return {'order_id': order.id}, 201

    except PaymentError as e:
        ORDER_COUNTER.labels(status='failed', product_category=category).inc()
        PAYMENT_FAILURES.labels(reason='payment_declined', gateway='stripe').inc()
        return {'error': str(e)}, 402

# Expose metrics on /metrics endpoint
app.wsgi_app = DispatcherMiddleware(app.wsgi_app, {
    '/metrics': make_wsgi_app()
})
```

### Go Application

```go
// main.go - Go app with Prometheus metrics
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    ordersProcessed = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "orders_processed_total",
            Help: "Total orders processed",
        },
        []string{"status", "category"},
    )

    orderDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "order_duration_seconds",
            Help:    "Order processing duration",
            Buckets: prometheus.DefBuckets,
        },
        []string{"category"},
    )

    activeConnections = promauto.NewGauge(prometheus.GaugeOpts{
        Name: "db_active_connections",
        Help: "Active database connections",
    })
)

func main() {
    // Expose metrics endpoint
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

## Step 2: Configure Prometheus to Scrape Custom Metrics

### Using ServiceMonitor

```yaml
# service-monitor.yaml - Prometheus ServiceMonitor for custom app metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-service-metrics
  namespace: cattle-monitoring-system
  labels:
    # Must match Prometheus' serviceMonitorSelector
    release: rancher-monitoring
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      # Select services with this label
      app: order-service
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
      honorLabels: true
      # Add relabeling if needed
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_label_version]
          targetLabel: version
```

Ensure your Service has matching labels:

```yaml
# order-service.yaml - Service with metrics annotations
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service  # Matches ServiceMonitor selector
spec:
  selector:
    app: order-service
  ports:
    - name: http      # Port name used in ServiceMonitor
      port: 8080
      targetPort: 8080
```

### Using PodMonitor

```yaml
# pod-monitor.yaml - PodMonitor for pods without services
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: batch-processor-metrics
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      app: batch-processor
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

### Using Annotations (Simple Approach)

For pods where you can't modify the ServiceMonitor:

```yaml
# annotated-pod.yaml - Pod with Prometheus scrape annotations
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-service
  namespace: production
spec:
  template:
    metadata:
      annotations:
        # These annotations are used by Prometheus automatic discovery
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
```

## Step 3: Deploy Custom Exporters

For applications that can't expose Prometheus metrics directly:

```yaml
# json-exporter.yaml - Convert JSON API to Prometheus metrics
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-json-exporter
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: json-exporter
          image: prometheuscommunity/json-exporter:v0.6.0
          args:
            - --config.file=/etc/json-exporter/config.yml
          ports:
            - containerPort: 7979
          volumeMounts:
            - name: config
              mountPath: /etc/json-exporter
      volumes:
        - name: config
          configMap:
            name: json-exporter-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: json-exporter-config
  namespace: production
data:
  config.yml: |
    modules:
      default:
        metrics:
          - name: orders_pending
            type: value
            path: '{.pending_orders}'
            help: Number of pending orders
          - name: revenue_today
            type: value
            path: '{.revenue.today}'
            help: Revenue generated today in USD
        body:
          content: ""
        headers:
          Authorization: "Bearer your-api-token"
        http_method: GET
```

## Step 4: Create Prometheus Recording Rules

```yaml
# recording-rules.yaml - Pre-computed metrics for performance
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: business-metrics-rules
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  groups:
    - name: business-metrics
      interval: 1m
      rules:
        # Order success rate (5-minute window)
        - record: orders:success_rate:5m
          expr: |
            sum(rate(orders_processed_total{status="success"}[5m]))
            /
            sum(rate(orders_processed_total[5m]))

        # Revenue per minute
        - record: revenue:per_minute:1m
          expr: |
            sum(increase(order_revenue_total[1m]))

        # P99 order latency by category
        - record: orders:latency_p99:5m
          expr: |
            histogram_quantile(0.99,
              sum(rate(order_processing_seconds_bucket[5m])) by (category, le)
            )
```

## Step 5: Create Grafana Dashboards for Business Metrics

```bash
# Import custom dashboard via Grafana API
curl -X POST \
  http://admin:admin@localhost:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d '{
    "dashboard": {
      "title": "Business Metrics",
      "panels": [
        {
          "title": "Orders Per Minute",
          "type": "stat",
          "targets": [{
            "expr": "sum(rate(orders_processed_total[1m])) * 60"
          }]
        },
        {
          "title": "Order Success Rate",
          "type": "gauge",
          "targets": [{
            "expr": "orders:success_rate:5m * 100"
          }]
        }
      ]
    },
    "overwrite": true
  }'
```

## Step 6: Set Up Alerting on Custom Metrics

```yaml
# business-alerts.yaml - Alerts based on custom business metrics
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: business-alerts
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  groups:
    - name: business-critical-alerts
      rules:
        - alert: LowOrderSuccessRate
          expr: orders:success_rate:5m < 0.95
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Order success rate dropped to {{ $value | humanizePercentage }}"

        - alert: HighPaymentFailureRate
          expr: rate(payment_failures_total[5m]) > 0.5
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Payment failure rate is {{ $value }} per second"
```

## Conclusion

Custom metrics transform Rancher monitoring from infrastructure-focused to business-focused observability. By instrumenting applications with the Prometheus client libraries and configuring ServiceMonitors, you create a feedback loop between business outcomes and system behavior. The key is to instrument what matters—error rates, latency percentiles, and business KPIs—and configure alerts that notify your team before customers notice degradation.
