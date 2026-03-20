# How to Configure Circuit Breakers in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Circuit Breakers, Istio, Resilience, Service Mesh

Description: Implement circuit breakers in Rancher using Istio's DestinationRule to prevent cascade failures and improve system resilience when downstream services degrade.

## Introduction

Circuit breakers prevent cascade failures in microservice architectures by temporarily blocking requests to unhealthy services. When a service is failing, instead of queuing up requests that will all fail, the circuit "opens" and fails fast, allowing the system to recover. Istio implements circuit breaking through the DestinationRule resource using two mechanisms: connection pool settings (to limit concurrent requests) and outlier detection (to eject unhealthy hosts).

## Prerequisites

- Rancher with Istio installed
- Services with Istio sidecar injection enabled
- kubectl with cluster-admin access

## Understanding Circuit Breaker States

The circuit breaker has three states:
- **Closed**: Normal operation, requests flow through
- **Open**: Service is failing, requests fail immediately
- **Half-Open**: Testing if service has recovered

## Step 1: Configure Connection Pool Limits

Connection pool limits prevent overwhelming downstream services:

```yaml
# connection-pool-limits.yaml - Limit concurrent connections
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-dr
  namespace: production
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        # Maximum concurrent TCP connections
        maxConnections: 100
        # TCP connection timeout
        connectTimeout: 30ms
        # Keep connections alive
        tcpKeepalive:
          time: 7200s
          interval: 75s
      http:
        # Maximum concurrent HTTP/1.1 requests
        http1MaxPendingRequests: 50
        # Maximum concurrent HTTP/2 requests per connection
        http2MaxRequests: 200
        # Maximum number of connection retries
        maxRequestsPerConnection: 100
        # Remove connections idle for more than 1 hour
        idleTimeout: 1h
```

## Step 2: Configure Outlier Detection

Outlier detection monitors host health and ejects failing instances:

```yaml
# outlier-detection.yaml - Eject unhealthy hosts automatically
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: database-service-dr
  namespace: production
spec:
  host: database-service
  trafficPolicy:
    outlierDetection:
      # Eject hosts with 5 consecutive 5xx errors
      consecutive5xxErrors: 5
      # Also eject on gateway errors
      consecutiveGatewayErrors: 3
      # How often to scan for unhealthy hosts
      interval: 10s
      # How long to keep host ejected
      baseEjectionTime: 30s
      # Maximum percentage of hosts that can be ejected
      maxEjectionPercent: 50
      # Minimum requests before ejection analysis starts
      minHealthPercent: 50
```

## Step 3: Comprehensive Circuit Breaker Configuration

Combine connection pool limits and outlier detection:

```yaml
# full-circuit-breaker.yaml - Production-grade circuit breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: orders-service-dr
  namespace: production
spec:
  host: orders-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 50
        connectTimeout: 5s
      http:
        http1MaxPendingRequests: 25
        http2MaxRequests: 100
        maxRetries: 3
    outlierDetection:
      consecutive5xxErrors: 5
      consecutiveGatewayErrors: 2
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 100
      # Minimum time between ejections
      splitExternalLocalOriginErrors: true
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        connectionPool:
          http:
            # Subset-specific override - more connections for v1
            http2MaxRequests: 150
```

## Step 4: Deploy a Test Scenario

Use Fortio to load test and trigger circuit breaking:

```bash
# Deploy Fortio load testing tool
kubectl apply -n production -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortio-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fortio
  template:
    metadata:
      labels:
        app: fortio
    spec:
      containers:
        - name: fortio
          image: fortio/fortio:latest
          command:
            - /usr/bin/fortio
          args:
            - server
          ports:
            - containerPort: 8080
EOF

# Generate traffic to test circuit breaker
kubectl exec -n production deployment/fortio-deploy -- \
  fortio load \
  -c 10 \            # 10 concurrent connections
  -qps 100 \         # 100 queries per second
  -t 30s \           # Run for 30 seconds
  http://payment-service:8080/api/payment
```

## Step 5: Monitor Circuit Breaker Metrics

```bash
# Query Prometheus for circuit breaker metrics
kubectl exec -n istio-system deployment/prometheus -- \
  curl -s 'http://localhost:9090/api/v1/query?query=sum(envoy_cluster_upstream_rq_pending_overflow)by(cluster_name)' | \
  jq '.data.result[] | {service: .metric.cluster_name, overflow_count: .value[1]}'

# Check ejected hosts
kubectl exec -n istio-system deployment/prometheus -- \
  curl -s 'http://localhost:9090/api/v1/query?query=sum(envoy_cluster_outlier_detection_ejections_active)by(cluster_name)' | \
  jq '.data.result[] | {service: .metric.cluster_name, ejected_hosts: .value[1]}'

# Check upstream request timeouts
kubectl exec -n istio-system deployment/prometheus -- \
  curl -s 'http://localhost:9090/api/v1/query?query=rate(envoy_cluster_upstream_rq_timeout[5m])' | \
  jq '.data.result'
```

## Step 6: Create Circuit Breaker Alerts

```yaml
# circuit-breaker-alerts.yaml - Prometheus alerts for circuit breaking
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: circuit-breaker-alerts
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  groups:
    - name: circuit-breaker
      rules:
        # Alert when circuit breaker is tripping
        - alert: CircuitBreakerOpen
          expr: |
            sum(rate(envoy_cluster_upstream_rq_pending_overflow[5m])) by (cluster_name) > 10
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "Circuit breaker tripping for {{ $labels.cluster_name }}"
            description: "{{ $value }} requests per second being rejected by circuit breaker"

        # Alert on high host ejection rate
        - alert: HighOutlierEjectionRate
          expr: |
            sum(envoy_cluster_outlier_detection_ejections_active) by (cluster_name) > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Hosts ejected by outlier detection in {{ $labels.cluster_name }}"
```

## Step 7: Implement Application-Level Circuit Breaker

For application-level circuit breaking, use Resilience4j in Java or similar:

```yaml
# app-with-circuit-breaker.yaml - Application with built-in circuit breaker
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: frontend
          image: registry.example.com/frontend:v1.0
          env:
            # Resilience4j circuit breaker configuration
            - name: RESILIENCE4J_CIRCUITBREAKER_INSTANCES_BACKEND_FAILURE_RATE_THRESHOLD
              value: "50"
            - name: RESILIENCE4J_CIRCUITBREAKER_INSTANCES_BACKEND_SLOW_CALL_RATE_THRESHOLD
              value: "70"
            - name: RESILIENCE4J_CIRCUITBREAKER_INSTANCES_BACKEND_SLOW_CALL_DURATION_THRESHOLD
              value: "2s"
            - name: RESILIENCE4J_CIRCUITBREAKER_INSTANCES_BACKEND_WAIT_DURATION_IN_OPEN_STATE
              value: "60s"
```

## Step 8: Verify Circuit Breaker Behavior

```bash
# Check Envoy statistics for the service
kubectl exec -n production -c istio-proxy deployment/frontend-app -- \
  curl -s localhost:15000/stats | grep "payment_service.*overflow\|ejection"

# Check outlier detection status
kubectl exec -n production -c istio-proxy deployment/frontend-app -- \
  curl -s localhost:15000/clusters | grep "outlier_detection"

# View Envoy cluster configuration
kubectl exec -n production -c istio-proxy deployment/frontend-app -- \
  curl -s localhost:15000/config_dump | jq '.configs[] | select(.["@type"] | contains("ClusterDiscoveryService"))'
```

## Conclusion

Circuit breakers are essential for building resilient microservice architectures in Rancher. Istio's DestinationRule provides fine-grained control over both connection pool limits (preventing resource exhaustion) and outlier detection (automatically ejecting unhealthy hosts). Combine circuit breakers with retry policies, timeouts, and bulkhead patterns for a comprehensive resilience strategy. Always monitor circuit breaker metrics and set up alerts to detect when services are degrading before they cause user-visible failures.
