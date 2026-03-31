# How to Choose Between Dapr Service Invocation and Direct HTTP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Service Invocation, HTTP, Microservice, Architecture

Description: Understand when to use Dapr service invocation versus direct HTTP calls, covering retries, observability, security, and performance trade-offs.

---

Dapr service invocation and direct HTTP are both valid for inter-service communication, but they offer very different capabilities. Choosing the wrong approach can leave you without retries, tracing, or mTLS - or add unnecessary latency when neither is needed.

## What Dapr Service Invocation Provides

When a service calls another via the Dapr sidecar, it gains automatic retries, distributed tracing, mTLS encryption, access control, and name-based service discovery without a service registry.

```bash
# Dapr service invocation - call via sidecar
curl http://localhost:3500/v1.0/invoke/inventory-service/method/stock/item-123

# Direct HTTP - call the pod IP or DNS directly
curl http://inventory-service.default.svc.cluster.local:8080/stock/item-123
```

## When to Use Dapr Service Invocation

Use Dapr service invocation for service-to-service calls that need observability and resilience:

```python
import requests

# Good use case: inter-service call with retries and tracing built in
def get_stock(item_id: str) -> dict:
    url = f"http://localhost:3500/v1.0/invoke/inventory-service/method/stock/{item_id}"
    response = requests.get(url, timeout=5)
    response.raise_for_status()
    return response.json()
```

Apply a resiliency policy to add retries transparently:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: service-resiliency
spec:
  policies:
    retries:
      defaultRetry:
        policy: exponential
        maxRetries: 3
        maxInterval: 10s
  targets:
    apps:
      inventory-service:
        outbound:
          retry: defaultRetry
```

## When to Use Direct HTTP

Direct HTTP is appropriate for calls to external services, third-party APIs, or when you control the retry logic yourself:

```go
package main

import (
    "net/http"
    "time"
)

// Direct HTTP: calling an external payment gateway
func chargePayment(token string, amount float64) error {
    client := &http.Client{Timeout: 10 * time.Second}
    req, _ := http.NewRequest("POST", "https://api.stripe.com/v1/charges", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    return nil
}
```

## Performance Comparison

Dapr service invocation adds one network hop through the sidecar. For latency-sensitive hot paths, measure the overhead:

```bash
# Benchmark direct HTTP
ab -n 1000 -c 10 http://inventory-service.default.svc.cluster.local:8080/health

# Benchmark via Dapr
ab -n 1000 -c 10 http://localhost:3500/v1.0/invoke/inventory-service/method/health
```

Typical overhead is 1-5ms per call - acceptable for most use cases, but relevant for tight loops.

## Security Considerations

Dapr service invocation enforces mTLS between sidecars automatically. Direct HTTP requires you to provision and rotate certificates manually:

```yaml
# Dapr access control - restrict which services can call each other
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  accessControl:
    defaultAction: deny
    trustDomain: "public"
    policies:
    - appId: checkout-service
      defaultAction: deny
      trustDomain: "public"
      namespace: "default"
      operations:
      - name: /reserve
        httpVerb: ["POST"]
        action: allow
```

## Decision Summary

| Criteria | Dapr Service Invocation | Direct HTTP |
|---|---|---|
| Retries | Built-in | Manual |
| Tracing | Automatic | Manual instrumentation |
| mTLS | Automatic | Manual cert management |
| External APIs | Not suitable | Correct choice |
| Latency | +1-5ms | Minimal |

## Summary

Choose Dapr service invocation for internal service-to-service calls where you need observability, retries, and mTLS without writing boilerplate code. Use direct HTTP for external API calls or cases where you already have a purpose-built HTTP client with custom retry logic. The Dapr overhead is small enough that the default should be Dapr invocation for all internal traffic.
