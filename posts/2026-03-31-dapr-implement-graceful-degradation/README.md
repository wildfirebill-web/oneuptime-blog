# How to Implement Graceful Degradation with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Resilience, Microservice, Graceful Degradation, Circuit Breaker

Description: Implement graceful degradation in Dapr-enabled microservices so your application stays functional when dependencies are slow or unavailable.

---

## What Is Graceful Degradation?

Graceful degradation means your application continues to serve users at reduced functionality rather than failing entirely when a dependency is unavailable. Dapr's resiliency policies and application-level fallbacks work together to achieve this.

## Configuring Circuit Breakers for Degradation

A circuit breaker trips after repeated failures and prevents cascading load on a failing service. Once open, Dapr returns an error immediately, and your app can serve a degraded response:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: graceful-degradation
  namespace: default
spec:
  policies:
    retries:
      quickRetry:
        policy: constant
        duration: 500ms
        maxRetries: 2
    circuitBreakers:
      recommendationCB:
        maxRequests: 1
        interval: 15s
        timeout: 60s
        trip: consecutiveFailures >= 5
  targets:
    apps:
      recommendation-service:
        retry: quickRetry
        circuitBreaker: recommendationCB
```

```bash
kubectl apply -f graceful-degradation.yaml
```

## Returning Degraded Responses in Python

```python
import os
import requests

DAPR_HTTP_PORT = os.getenv("DAPR_HTTP_PORT", "3500")

def get_recommendations(user_id: str) -> dict:
    url = f"http://localhost:{DAPR_HTTP_PORT}/v1.0/invoke/recommendation-service/method/recommend/{user_id}"
    try:
        resp = requests.get(url, timeout=2)
        resp.raise_for_status()
        return resp.json()
    except Exception as e:
        print(f"Recommendation service degraded: {e}")
        # Return generic recommendations as fallback
        return {
            "userId": user_id,
            "items": ["popular-item-1", "popular-item-2"],
            "degraded": True
        }
```

## Degraded State via Dapr State Store

Cache the last known good response in a Dapr state store so you can serve it during degradation:

```python
import json

def cache_recommendations(user_id: str, data: dict):
    url = f"http://localhost:{DAPR_HTTP_PORT}/v1.0/state/statestore"
    payload = [{"key": f"recs-{user_id}", "value": data}]
    requests.post(url, json=payload)

def get_cached_recommendations(user_id: str) -> dict:
    url = f"http://localhost:{DAPR_HTTP_PORT}/v1.0/state/statestore/recs-{user_id}"
    resp = requests.get(url)
    if resp.status_code == 200 and resp.text:
        return resp.json()
    return {"userId": user_id, "items": [], "degraded": True}
```

## Using Dapr Pub/Sub for Async Degradation

For write operations, publish events to a queue so they are not lost when a service is down:

```python
def place_order_degraded(order: dict):
    url = f"http://localhost:{DAPR_HTTP_PORT}/v1.0/publish/pubsub/orders-pending"
    requests.post(url, json=order)
    return {"status": "queued", "message": "Order will be processed shortly"}
```

## Monitoring Degradation Events

Add structured logs so you can track degradation frequency:

```bash
kubectl logs -l app=storefront -c daprd --tail=100 | grep -i "circuit\|fallback\|degraded"
```

## Summary

Graceful degradation with Dapr combines circuit breakers defined in Resiliency CRDs with application-level fallback logic. Storing last-known-good state in a Dapr state store and queuing writes via pub/sub ensures your service remains useful even when dependencies fail. Monitor circuit breaker state to know when to scale or fix downstream services.
