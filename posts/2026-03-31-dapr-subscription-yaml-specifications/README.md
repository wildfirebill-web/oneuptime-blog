# How to Write Dapr Subscription YAML Specifications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Subscription, YAML, Pub/Sub, Event-Driven

Description: Learn how to write Dapr Subscription YAML files including declarative routing, dead letter topics, scoping, and bulk subscription configuration.

---

## Overview

Dapr Subscription YAML files define how topics are routed to application endpoints declaratively, without requiring your app to expose a `/dapr/subscribe` route. Declarative subscriptions are defined as Kubernetes custom resources or local YAML files and give you routing rules, dead letter topics, and scoping in one place.

## Subscription YAML Structure

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: <subscription-name>
  namespace: <kubernetes-namespace>
spec:
  pubsubname: <pubsub-component-name>
  topic: <topic-name>
  routes:
    default: <default-route-path>
    rules:
      - match: <CEL-expression>
        path: <route-path>
  deadLetterTopic: <dead-letter-topic-name>
  bulkSubscribe:
    enabled: false
    maxMessagesCount: 100
    maxAwaitDurationMs: 1000
scopes:
  - <app-id>
```

## Basic Subscription

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: orders-subscription
  namespace: default
spec:
  pubsubname: pubsub
  topic: orders
  routes:
    default: /handle-order
scopes:
  - order-processor
```

## Content-Based Routing

Route messages to different endpoints based on message content using Common Expression Language (CEL):

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: orders-routing-subscription
spec:
  pubsubname: pubsub
  topic: orders
  routes:
    rules:
      - match: event.data.priority == "urgent"
        path: /handle-urgent-order
      - match: event.data.type == "international"
        path: /handle-international-order
    default: /handle-standard-order
scopes:
  - order-processor
```

## Dead Letter Topic

Route messages that fail all retry attempts to a separate dead letter topic:

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: payments-subscription
spec:
  pubsubname: pubsub
  topic: payment-events
  routes:
    default: /process-payment
  deadLetterTopic: payment-events-dead-letter
scopes:
  - payment-service
```

Handle dead letter messages separately:

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: payment-dlq-subscription
spec:
  pubsubname: pubsub
  topic: payment-events-dead-letter
  routes:
    default: /handle-failed-payment
scopes:
  - payment-service
```

## Bulk Subscription

Process messages in batches to improve throughput:

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: events-bulk-subscription
spec:
  pubsubname: pubsub
  topic: analytics-events
  routes:
    default: /handle-events-batch
  bulkSubscribe:
    enabled: true
    maxMessagesCount: 50
    maxAwaitDurationMs: 500
scopes:
  - analytics-service
```

Your handler receives a batch of events:

```javascript
app.post("/handle-events-batch", (req, res) => {
  const { entries } = req.body;
  const results = entries.map(entry => {
    processEvent(entry.event);
    return { entryId: entry.entryId, status: "SUCCESS" };
  });
  res.json({ statuses: results });
});
```

## Self-Hosted File Location

Place subscription YAML files in the same components directory:

```bash
# Default location
~/.dapr/components/orders-subscription.yaml

# Custom location
dapr run --resources-path ./components -- node app.js
```

## Applying to Kubernetes

```bash
kubectl apply -f orders-subscription.yaml
kubectl get subscriptions -n default
```

## Subscription vs Programmatic Subscribe

| Approach | Pros | Cons |
|---|---|---|
| YAML Subscription | Declarative, GitOps-friendly | Requires file/CRD management |
| Programmatic `/dapr/subscribe` | Self-contained in app code | Routing logic in app |

## Summary

Dapr Subscription YAML files externalize pub/sub routing configuration, making it easy to manage topic subscriptions through GitOps workflows without modifying application code. Content-based routing, dead letter topics, and bulk subscribe options are all expressible in a single YAML file, giving operators control over message routing independent of application deployments.
