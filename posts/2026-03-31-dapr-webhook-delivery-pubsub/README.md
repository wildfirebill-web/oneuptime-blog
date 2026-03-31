# How to Implement Webhook Delivery with Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Webhook, HTTP, Integration

Description: Learn how to implement reliable webhook delivery using Dapr pub/sub to fan out events to registered external endpoints with retries and delivery tracking.

---

## Why Use Dapr for Webhook Delivery?

Webhooks require reliable delivery to potentially unreliable external HTTP endpoints. Dapr pub/sub provides built-in retry policies, dead letter queues, and at-least-once delivery guarantees - handling the hard parts of webhook infrastructure.

## Architecture

1. Business event occurs (order placed, payment succeeded).
2. A Dapr topic receives the event.
3. A webhook dispatcher service subscribes and delivers to registered endpoint URLs.
4. Failed deliveries are retried with exponential backoff.

## Configure the Pub/Sub Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: webhook-pubsub
  namespace: production
spec:
  type: pubsub.redis
  version: v1
  metadata:
    - name: redisHost
      value: "redis:6379"
```

## Webhook Registration Store

Store registered webhooks in Dapr state:

```javascript
const { DaprClient } = require('@dapr/dapr');
const client = new DaprClient();

async function registerWebhook(customerId, eventType, targetUrl, secret) {
  const webhookId = require('crypto').randomUUID();
  await client.state.save('state-store', [{
    key: `webhook:${customerId}:${eventType}:${webhookId}`,
    value: { webhookId, customerId, eventType, targetUrl, secret, active: true }
  }]);
  return webhookId;
}
```

## Webhook Dispatcher Service

```javascript
const { DaprServer } = require('@dapr/dapr');
const server = new DaprServer();
const axios = require('axios');
const crypto = require('crypto');

await server.pubsub.subscribe('webhook-pubsub', 'order.placed', async (event) => {
  // Find all registered webhooks for this event type
  const webhooks = await getWebhooksForEvent(event.customerId, 'order.placed');

  await Promise.allSettled(
    webhooks.map(webhook => deliverWebhook(webhook, event))
  );
});

async function deliverWebhook(webhook, payload) {
  const body = JSON.stringify(payload);
  const signature = crypto
    .createHmac('sha256', webhook.secret)
    .update(body)
    .digest('hex');

  const response = await axios.post(webhook.targetUrl, payload, {
    headers: {
      'Content-Type': 'application/json',
      'X-Webhook-Signature': `sha256=${signature}`,
      'X-Webhook-Event': payload.eventType,
      'X-Webhook-Id': require('crypto').randomUUID()
    },
    timeout: 10_000
  });

  await logDelivery(webhook.webhookId, response.status, 'delivered');
  return response;
}
```

## Retry Logic with Dead Letter Queue

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: webhook-resiliency
spec:
  policies:
    retries:
      webhook-retry:
        policy: exponential
        maxRetries: 5
        maxInterval: 60s
  targets:
    apps:
      webhook-dispatcher:
        retry: webhook-retry
```

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: webhook-subscription
spec:
  topic: order.placed
  routes:
    default: /webhooks/deliver
  pubsubname: webhook-pubsub
  deadLetterTopic: webhook-failed
```

## Tracking Delivery Status

```javascript
async function logDelivery(webhookId, statusCode, status) {
  await client.state.save('state-store', [{
    key: `delivery:${webhookId}:${Date.now()}`,
    value: { webhookId, statusCode, status, deliveredAt: new Date().toISOString() },
    metadata: { ttlInSeconds: '604800' }  // Keep for 7 days
  }]);
}
```

## Summary

Dapr pub/sub combined with resiliency policies provides a production-grade webhook delivery system. HMAC signatures ensure payload authenticity, retry policies handle transient failures, and dead letter topics capture permanently failed deliveries for manual investigation or customer notification.
