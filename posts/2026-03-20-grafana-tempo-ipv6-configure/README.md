# How to Configure Grafana Tempo with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Grafana, Tempo, IPv6, Tracing, Observability, OpenTelemetry

Description: A guide to configuring Grafana Tempo to accept traces over IPv6 and querying IPv6 trace data from Grafana's Explore view.

Grafana Tempo is a distributed tracing backend. Configuring it for IPv6 allows traces from IPv6-addressed services to be collected and correlated with IPv6 network events, providing complete observability for IPv6 workloads.

## Step 1: Configure Tempo to Listen on IPv6

Edit `tempo.yaml` to bind all listening ports to IPv6:

```yaml
# tempo.yaml - Tempo configuration with IPv6 listeners
server:
  # Listen on all interfaces (both IPv4 and IPv6)
  # Use :: to bind all interfaces; [::] notation in the config
  http_listen_address: ""      # Empty = all interfaces (both IPv4/IPv6)
  http_listen_port: 3200
  grpc_listen_address: ""
  grpc_listen_port: 9095

# Distributor accepts traces from OpenTelemetry Collector
distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          # Listen on all interfaces including IPv6
          endpoint: "[::]:4317"
        http:
          endpoint: "[::]:4318"
    jaeger:
      protocols:
        thrift_http:
          endpoint: "[::]:14268"
    zipkin:
      endpoint: "[::]:9411"

# Storage backend
storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces

compactor:
  compaction:
    block_retention: 48h
```

## Step 2: Start Tempo with IPv6 Config

```bash
# Start Tempo with the IPv6 configuration
tempo -config.file=tempo.yaml

# Verify it's listening on IPv6
ss -6 -tlnp | grep tempo
# Expected: *:3200 (http), *:9095 (grpc), *:4317 (otlp-grpc)
```

## Step 3: Configure OpenTelemetry Collector to Send to Tempo over IPv6

```yaml
# otel-collector-config.yaml - Send traces to Tempo via IPv6
exporters:
  otlp:
    # Use IPv6 address for Tempo
    endpoint: "[2001:db8::tempo]:4317"
    tls:
      insecure: true    # For development; use proper TLS in production

  # Or using the Tempo HTTP endpoint
  otlphttp:
    endpoint: "http://[2001:db8::tempo]:4318"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```

## Step 4: Configure Grafana to Use Tempo Data Source over IPv6

In Grafana's UI or via provisioning:

```yaml
# grafana/provisioning/datasources/tempo.yaml - Tempo data source with IPv6
apiVersion: 1

datasources:
  - name: Tempo
    type: tempo
    # Connect to Tempo via IPv6
    url: "http://[2001:db8::tempo]:3200"
    access: proxy
    isDefault: false
    jsonData:
      # Enable trace to logs correlation
      tracesToLogs:
        datasourceUid: "loki"
      # Enable service graph
      serviceMap:
        datasourceUid: "prometheus"
```

## Step 5: Query IPv6 Trace Data in Grafana

In the Grafana Explore view with Tempo selected, use TraceQL to find traces from IPv6 services:

```
# Find traces from services running on IPv6 addresses
{ .net.peer.ip =~ ".*:.*" }

# Find slow IPv6 HTTP requests (>1s)
{ .http.url =~ ".*\\[.*\\].*" && duration > 1s }

# Find traces with IPv6 source IPs
{ .network.source.ip =~ "[0-9a-f:]+" && span.kind = server }
```

## Step 6: Kubernetes Deployment with IPv6

```yaml
# tempo-deployment.yaml - Kubernetes deployment for Tempo with IPv6
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tempo
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: tempo
  template:
    spec:
      containers:
        - name: tempo
          image: grafana/tempo:latest
          args:
            - -config.file=/etc/tempo/tempo.yaml
          ports:
            - name: http
              containerPort: 3200
              protocol: TCP
            - name: otlp-grpc
              containerPort: 4317
              protocol: TCP
```

## Verify Trace Ingestion

```bash
# Send a test trace to Tempo via IPv6
curl -X POST "http://[2001:db8::tempo]:4318/v1/traces" \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans": []}'

# Query for recent traces
curl "http://[2001:db8::tempo]:3200/api/search?limit=5" | jq '.traces[].rootTraceName'
```

Configuring Tempo to listen on IPv6 ensures that traces from IPv6-addressed services are collected without any special routing, providing full observability coverage for dual-stack and IPv6-only deployments.
