# How to Configure Retry Policies with Service Mesh in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Service Mesh, Retry Policies, Istio, Resilience

Description: Configure intelligent retry policies for your microservices in Rancher using Istio to handle transient failures and improve application resilience.

## Introduction

Retry policies automatically retry failed requests to improve resilience against transient failures. In a microservices architecture, network hiccups, temporary service overload, or rolling deployments can cause intermittent failures. Service mesh retry policies handle these gracefully without requiring changes to application code. This guide covers configuring retry policies using Istio in Rancher-managed clusters.

## Prerequisites

- Rancher with Istio installed
- Services with Istio sidecars injected
- kubectl with cluster-admin access

## Understanding Retry Semantics

Before configuring retries, understand the types of failures:
- **Connection failures**: TCP connection refused or timed out
- **5xx errors**: Server-side errors (502, 503, 504)
- **Gateway errors**: Upstream connection resets
- **Retriable errors**: Idempotent operations that can safely be retried

**Important**: Only configure retries for idempotent operations (GET, PUT, DELETE). Never retry non-idempotent POST requests that create resources, as this can cause duplicate operations.

## Step 1: Basic Retry Policy in Istio VirtualService

```yaml
# basic-retry.yaml - Simple retry policy

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service-vs
  namespace: production
spec:
  hosts:
    - payment-service
  http:
    - name: payment-retry-policy
      route:
        - destination:
            host: payment-service
            port:
              number: 8080
      retries:
        # Number of retry attempts
        attempts: 3
        # Timeout per retry attempt
        perTryTimeout: 2s
        # Retry on these conditions
        retryOn: "5xx,reset,connect-failure,retriable-4xx"
```

## Step 2: Advanced Retry Configuration

```yaml
# advanced-retry.yaml - Per-route retry with different settings
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-gateway-vs
  namespace: production
spec:
  hosts:
    - api-gateway
  http:
    # Read endpoints - safe to retry
    - name: read-operations
      match:
        - headers:
            method:
              exact: GET
      route:
        - destination:
            host: api-gateway
      retries:
        attempts: 5
        perTryTimeout: 3s
        # Comprehensive retry conditions for read operations
        retryOn: "5xx,reset,connect-failure,retriable-4xx,gateway-error"
        # Retry headers (for tracking retries)
        retryRemoteLocalities: true

    # Write endpoints - be careful with retries
    - name: write-operations
      route:
        - destination:
            host: api-gateway
      retries:
        # Fewer retries for writes to avoid duplicates
        attempts: 2
        perTryTimeout: 5s
        # Only retry on connection-level failures, not application errors
        retryOn: "connect-failure,reset"

      # Overall timeout including retries
      timeout: 15s
```

## Step 3: Retry Policy with Hedging

Hedge requests allow parallel retry attempts:

```yaml
# hedged-retry.yaml - Hedge requests for latency-sensitive services
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: search-service-vs
  namespace: production
spec:
  hosts:
    - search-service
  http:
    - name: search-with-hedge
      route:
        - destination:
            host: search-service
      timeout: 500ms  # Short overall timeout
      retries:
        attempts: 3
        perTryTimeout: 200ms  # Tight per-try timeout triggers hedge
        retryOn: "5xx,connect-failure,reset"
```

## Step 4: Retry with Circuit Breaker Integration

```yaml
# retry-with-cb.yaml - Retry policy that respects circuit breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend-dr
  namespace: production
spec:
  host: backend-service
  trafficPolicy:
    # Circuit breaker settings
    outlierDetection:
      # Eject hosts with 50% error rate
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 100
    connectionPool:
      http:
        # Limit pending requests to prevent overload
        http1MaxPendingRequests: 100
        h2UpgradePolicy: UPGRADE
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend-vs
  namespace: production
spec:
  hosts:
    - backend-service
  http:
    - route:
        - destination:
            host: backend-service
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: "5xx,gateway-error,connect-failure,retriable-4xx"
```

## Step 5: Test Retry Policies

Deploy a fault injection to test your retry configuration:

```yaml
# fault-injection-test.yaml - Inject 50% failures to test retries
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend-fault-test
  namespace: production
spec:
  hosts:
    - backend-service
  http:
    - fault:
        abort:
          percentage:
            value: 50  # Fail 50% of requests
          httpStatus: 503
      route:
        - destination:
            host: backend-service
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: "5xx"
```

```bash
# Test the retry policy is working
for i in {1..20}; do
  kubectl exec -n production deployment/frontend -- \
    curl -s -o /dev/null -w "%{http_code}\n" \
    http://backend-service:8080/health
done
# Should see mostly 200s despite 50% fault injection
```

## Step 6: Monitor Retry Metrics

```bash
# Query Prometheus for retry metrics
kubectl exec -n istio-system deployment/prometheus -- \
  curl -s 'http://localhost:9090/api/v1/query?query=sum(rate(envoy_cluster_upstream_rq_retry[5m]))by(cluster_name)' | \
  jq '.data.result[] | {service: .metric.cluster_name, retries_per_second: .value[1]}'

# Check overall retry rate
kubectl exec -n istio-system deployment/prometheus -- \
  curl -s 'http://localhost:9090/api/v1/query?query=sum(rate(envoy_cluster_upstream_rq_retry[5m]))/sum(rate(envoy_cluster_upstream_rq_total[5m]))' | \
  jq '.data.result[0].value[1]'
```

## Step 7: Configure Retry with Rancher Fleet

```yaml
# fleet-values.yaml - Fleet bundle with retry configuration
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app-vs
spec:
  hosts:
    - app-service
  http:
    - route:
        - destination:
            host: app-service
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: "5xx,reset"
```

## Best Practices

```yaml
# best-practices.yaml - Production-ready retry configuration
# 1. Set overall timeout <= (attempts * perTryTimeout)
# 2. Use exponential backoff (Istio handles this automatically)
# 3. Monitor retry rates - high retries indicate underlying issues
# 4. Only retry idempotent operations
# 5. Combine with circuit breakers for defense in depth

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: production-service-vs
spec:
  hosts:
    - production-service
  http:
    - name: production-retry-policy
      route:
        - destination:
            host: production-service
      # Conservative retry configuration
      retries:
        attempts: 3              # Max 3 attempts
        perTryTimeout: 3s        # 3s per attempt
        retryOn: "5xx,reset,connect-failure"
      timeout: 10s               # Total timeout: 10s
```

## Conclusion

Retry policies are a critical resilience pattern for microservices running in Rancher-managed clusters. Istio's VirtualService makes it easy to configure intelligent retries without changing application code. Remember to carefully consider which operations are safe to retry, monitor your retry rates to detect underlying issues, and combine retries with circuit breakers to prevent cascade failures. A well-configured retry policy can dramatically improve user experience during transient infrastructure issues.
