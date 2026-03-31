# How to Implement Ambassador Pattern with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Ambassador, Pattern, Proxy, Microservice

Description: Learn how Dapr implements the ambassador pattern to offload network tasks like retries, circuit breaking, and protocol translation to a sidecar proxy.

---

## Overview

The Ambassador pattern deploys a proxy (ambassador) alongside a service to handle network-related concerns: retry logic, circuit breaking, protocol translation, and connection pooling. Dapr's sidecar acts as an ambassador, handling these concerns so your application code stays focused on business logic.

## Ambassador vs Sidecar

While closely related, the ambassador pattern specifically focuses on network proxying concerns. Dapr's ambassador responsibilities include:

- Retrying failed outbound requests
- Circuit breaking for unstable services
- Protocol translation (HTTP to gRPC)
- mTLS establishment
- Load balancing across service instances

## Configuring Resiliency as Ambassador Behavior

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: ambassador-resiliency
  namespace: default
spec:
  policies:
    timeouts:
      fast: 2s
      slow: 10s
    retries:
      standard:
        policy: exponential
        duration: 1s
        maxInterval: 10s
        maxRetries: 3
      aggressive:
        policy: constant
        duration: 500ms
        maxRetries: 5
    circuitBreakers:
      payment-cb:
        maxRequests: 2
        interval: 10s
        timeout: 30s
        trip: consecutiveFailures >= 3
  targets:
    apps:
      payment-service:
        timeout: fast
        retry: standard
        circuitBreaker: payment-cb
      catalog-service:
        timeout: slow
        retry: aggressive
```

## Application Code - Ambassador Handles Complexity

Your application code simply invokes services. The ambassador (Dapr sidecar) handles all retry, timeout, and circuit breaking logic:

```go
package main

import (
    "context"
    "log"
    dapr "github.com/dapr/go-sdk/client"
)

type OrderService struct {
    client dapr.Client
}

func (svc *OrderService) PlaceOrder(orderID, customerID string, amount float64) error {
    ctx := context.Background()

    // Dapr sidecar acts as ambassador:
    // - Retries on transient failures
    // - Opens circuit breaker after repeated failures
    // - Enforces timeout
    // - Encrypts with mTLS
    result, err := svc.client.InvokeMethodWithContent(
        ctx,
        "payment-service",
        "/charge",
        "POST",
        &dapr.DataContent{
            ContentType: "application/json",
            Data:        []byte(`{"orderId":"` + orderID + `","amount":` + fmt.Sprintf("%.2f", amount) + `}`),
        },
    )
    if err != nil {
        // Ambassador already retried; this is a real failure
        log.Printf("Payment failed after retries: %v", err)
        return err
    }

    log.Printf("Payment result: %s", result)
    return nil
}
```

## Protocol Translation

The ambassador translates HTTP to gRPC for services that require it:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: grpc-proxy
spec:
  type: middleware.http.grpc
  version: v1
  metadata:
    - name: serverStream
      value: "false"
```

For gRPC-only backend services, configure the target app:

```yaml
annotations:
  dapr.io/app-protocol: "grpc"
  dapr.io/app-port: "50051"
```

## Connection Pooling and Load Balancing

Dapr's ambassador maintains a connection pool to backend services. Name resolution handles load balancing:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: nameresolution
spec:
  type: nameresolution.kubernetes
  version: v1
```

Dapr uses Kubernetes DNS-based service discovery and distributes requests across healthy pod instances automatically.

## Observing Ambassador Behavior

View sidecar logs to see ambassador activity:

```bash
kubectl logs -l app=order-service -c daprd --tail=50
```

Check circuit breaker state via metrics:

```bash
kubectl port-forward svc/dapr-metrics 9090:9090
# Query: dapr_resiliency_count{app_id="order-service"}
```

## Summary

Dapr implements the ambassador pattern through its sidecar, transparently handling retries, circuit breaking, timeouts, mTLS, and protocol translation. Application code calls services via the Dapr sidecar without awareness of network reliability logic. Resiliency policies declared in YAML configure ambassador behavior per target service, making it easy to tune failure handling without code changes.
