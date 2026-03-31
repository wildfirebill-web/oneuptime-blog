# How to Secure Dapr Pub/Sub Topics with Scoping

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Security, Scoping, Topic

Description: Learn how to use Dapr pub/sub topic scoping to restrict which applications can publish to or subscribe from specific topics on a shared message broker.

---

## Overview

When multiple services share a single Dapr pub/sub component, any service can by default publish to or subscribe from any topic. Pub/sub scoping allows you to restrict which app IDs are permitted to publish and which are permitted to subscribe, enforcing least-privilege messaging without requiring separate brokers per service.

## How Pub/Sub Scoping Works

Scoping is configured at the component level using two metadata fields:

- `publishingScopes`: Defines which app IDs can publish to which topics
- `subscriptionScopes`: Defines which app IDs can subscribe to which topics
- `allowedTopics`: Restricts the entire component to a list of allowed topics

## Configuring Topic Scoping

Create a Redis Streams pub/sub component with scopes:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: orders-pubsub
  namespace: default
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis:6379"
  - name: allowedTopics
    value: "orders,payments,notifications"
  - name: publishingScopes
    value: "checkout-service=orders;payment-service=payments"
  - name: subscriptionScopes
    value: "inventory-service=orders;notification-service=orders,payments,notifications"
```

In this example:
- Only `checkout-service` can publish to the `orders` topic
- Only `payment-service` can publish to the `payments` topic
- `inventory-service` can only subscribe to `orders`
- `notification-service` can subscribe to all three topics

## Restricting with allowedTopics

The `allowedTopics` field prevents any service from using topics not on the list, even if they are not explicitly scoped:

```yaml
  - name: allowedTopics
    value: "orders,payments"
```

An attempt to publish to a topic called `debug-events` would be blocked.

## Testing Scoping Restrictions

Try publishing from an unauthorized app:

```bash
# From inventory-service (not in publishingScopes for orders)
curl -X POST http://localhost:3500/v1.0/publish/orders-pubsub/orders \
  -H "Content-Type: application/json" \
  -d '{"orderId": "123"}'
# Expected: 403 Forbidden
```

From the authorized publisher:

```bash
# From checkout-service (authorized to publish to orders)
curl -X POST http://localhost:3500/v1.0/publish/orders-pubsub/orders \
  -H "Content-Type: application/json" \
  -d '{"orderId": "123"}'
# Expected: 204 No Content
```

## Namespace-Level Isolation

For stronger isolation, deploy separate pub/sub components per namespace instead of relying solely on scopes:

```bash
# Tenant A gets its own component
kubectl apply -f pubsub-tenant-a.yaml -n tenant-a
# Tenant B gets its own component
kubectl apply -f pubsub-tenant-b.yaml -n tenant-b
```

## Summary

Dapr pub/sub topic scoping provides a simple declarative way to enforce least-privilege messaging on shared brokers. By configuring `publishingScopes`, `subscriptionScopes`, and `allowedTopics` in the component definition, you prevent unintended cross-service communication through message topics without additional infrastructure.
