# How to Understand Dapr Subscription Specification Format

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Subscription, YAML, Specification, Event

Description: Understand the Dapr subscription specification format, including declarative and programmatic subscriptions, routing rules, dead-letter topics, and scopes.

---

## What Is a Dapr Subscription?

A Dapr subscription tells the Dapr runtime which topics an application wants to receive messages from and how to route them to specific endpoints. Subscriptions can be defined declaratively (YAML) or programmatically (in code).

## Declarative Subscription Format

The declarative subscription YAML uses the `Subscription` kind:

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: order-subscription
spec:
  pubsubname: pubsub       # References a pubsub component
  topic: orders            # Topic name to subscribe to
  routes:
    default: /orders/process
  deadLetterTopic: orders-dlq
  scopes:
    - order-processor
```

## The `routes` Field

The `routes` field maps messages to HTTP endpoints in your application. You can use a simple `default` route or add CEL-based routing rules:

```yaml
routes:
  rules:
    - match: 'event.type == "order.created"'
      path: /orders/created
    - match: 'event.type == "order.cancelled"'
      path: /orders/cancelled
  default: /orders/fallback
```

The `match` expressions use the Common Expression Language (CEL) and evaluate against the CloudEvent envelope.

## Programmatic Subscription in Go

Register subscriptions in code for dynamic routing:

```go
func (s *Server) ListTopicSubscriptions(ctx context.Context,
    in *emptypb.Empty) (*runtimev1pb.ListTopicSubscriptionsResponse, error) {
    return &runtimev1pb.ListTopicSubscriptionsResponse{
        Subscriptions: []*runtimev1pb.TopicSubscription{
            {
                PubsubName: "pubsub",
                Topic:      "orders",
                Routes: &runtimev1pb.TopicRoutes{
                    Default: "/orders/process",
                },
            },
        },
    }, nil
}
```

## Dead-Letter Topics

When message processing fails, Dapr can forward messages to a dead-letter topic:

```yaml
spec:
  pubsubname: pubsub
  topic: orders
  routes:
    default: /orders/process
  deadLetterTopic: orders-failed
  bulkSubscribe:
    enabled: true
    maxMessagesCount: 100
    maxAwaitDurationMs: 1000
```

Subscribe to the dead-letter topic in a separate service for alerting or reprocessing.

## Bulk Subscribe Configuration

Dapr v1.11+ supports bulk subscribe to improve throughput:

```yaml
spec:
  pubsubname: pubsub
  topic: high-volume-events
  routes:
    default: /events/process
  bulkSubscribe:
    enabled: true
    maxMessagesCount: 250
    maxAwaitDurationMs: 500
```

Your endpoint will receive a batch of events in a single HTTP POST.

## Scoping Subscriptions

The `scopes` field restricts which app IDs receive the subscription:

```yaml
scopes:
  - inventory-service
  - warehouse-service
```

Without scopes, all apps with the subscription YAML in their path receive the subscription.

## Verifying Subscriptions

Check active subscriptions via the Dapr API:

```bash
curl http://localhost:3500/v1.0/metadata | jq '.subscriptions'
```

Or check the Dapr dashboard:

```bash
dapr dashboard -k -p 9999
```

## Summary

The Dapr subscription specification uses `apiVersion: dapr.io/v2alpha1` and supports declarative YAML or programmatic registration. Key fields include `routes` (with optional CEL-based rules), `deadLetterTopic`, `bulkSubscribe`, and `scopes`. Using route rules allows fine-grained message routing to different application endpoints based on CloudEvent attributes.
