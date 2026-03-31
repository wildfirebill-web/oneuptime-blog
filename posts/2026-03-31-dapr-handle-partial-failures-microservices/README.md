# How to Handle Partial Failures in Dapr Microservices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Microservice, Resilience, Fault Tolerance, Kubernetes

Description: Learn how to detect and handle partial failures in Dapr microservices using retries, timeouts, circuit breakers, and fallback strategies.

---

## Understanding Partial Failures

Partial failures occur when one or more services in a distributed system fail while others remain healthy. Without proper handling, a single failing dependency can cascade and bring down your entire application. Dapr provides built-in resiliency primitives to handle these scenarios gracefully.

## Defining a Resiliency Policy

Dapr resiliency policies are defined as Kubernetes CRDs. Create a policy that applies retries and timeouts to service invocations:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: partial-failure-policy
  namespace: default
spec:
  policies:
    retries:
      retryThree:
        policy: constant
        duration: 1s
        maxRetries: 3
    timeouts:
      shortTimeout: 2s
    circuitBreakers:
      serviceCB:
        maxRequests: 1
        interval: 10s
        timeout: 30s
        trip: consecutiveFailures >= 3
  targets:
    apps:
      order-service:
        retry: retryThree
        timeout: shortTimeout
        circuitBreaker: serviceCB
```

Apply the policy:

```bash
kubectl apply -f resiliency-policy.yaml
```

## Handling Failures in Application Code

Even with Dapr policies, your application should handle non-2xx responses gracefully. Here is a Node.js example using the Dapr HTTP client:

```javascript
const axios = require('axios');

async function callOrderService(orderId) {
  const daprPort = process.env.DAPR_HTTP_PORT || 3500;
  const url = `http://localhost:${daprPort}/v1.0/invoke/order-service/method/orders/${orderId}`;

  try {
    const response = await axios.get(url, { timeout: 3000 });
    return response.data;
  } catch (error) {
    if (error.response && error.response.status === 503) {
      // Service unavailable - return cached or default data
      console.warn('Order service unavailable, returning default');
      return { orderId, status: 'unknown', cached: true };
    }
    throw error;
  }
}
```

## Using Fallback Responses

When a downstream service is degraded, return a sensible default instead of propagating the error:

```javascript
async function getInventory(productId) {
  try {
    const result = await daprClient.invoker.invoke(
      'inventory-service',
      'stock',
      HttpMethod.GET
    );
    return result;
  } catch (err) {
    console.error(`Inventory service failed: ${err.message}`);
    // Fallback: assume sufficient stock to avoid blocking user
    return { productId, quantity: 999, fallback: true };
  }
}
```

## Observing Partial Failures with Metrics

Enable Dapr metrics to track retry counts and circuit breaker state:

```bash
kubectl port-forward svc/dapr-dashboard -n dapr-system 8080:8080
```

You can also query Prometheus metrics directly:

```bash
kubectl port-forward svc/prometheus -n monitoring 9090:9090
# Query: dapr_resiliency_count_total{namespace="default",name="partial-failure-policy"}
```

## Testing Partial Failure Scenarios

Use `kubectl` to simulate a failing service by scaling it to zero:

```bash
kubectl scale deployment order-service --replicas=0
# Watch your calling service handle the partial failure
kubectl logs -f deployment/checkout-service -c daprd
# Restore the service
kubectl scale deployment order-service --replicas=2
```

## Summary

Dapr provides retries, timeouts, and circuit breakers through Resiliency CRDs to handle partial failures in microservices. Combine these policies with application-level fallbacks to build truly resilient distributed systems. Monitoring Dapr metrics helps you observe and tune these policies under real load.
