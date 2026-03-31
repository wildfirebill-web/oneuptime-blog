# How to Use Dapr Pub/Sub for Request-Reply Patterns

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Request-Reply, Correlation, Microservice

Description: Learn how to implement request-reply messaging over Dapr pub/sub using correlation IDs and reply topics to achieve bidirectional communication asynchronously.

---

## When to Use Request-Reply over Pub/Sub

Dapr service invocation handles synchronous request-reply natively. However, when the responder may be slow, unavailable, or needs to process in the background, request-reply over pub/sub provides resilience with asynchronous correlation.

## The Request-Reply Pattern

1. Requester publishes a request message with a `correlationId` and a `replyTopic`
2. Responder subscribes, processes, and publishes the result to the `replyTopic`
3. Requester subscribes to the `replyTopic` and matches responses using `correlationId`

## Requester Implementation

```javascript
const { v4: uuidv4 } = require("uuid");

const pendingRequests = new Map();

async function sendRequest(payload, timeoutMs = 10000) {
  const correlationId = uuidv4();

  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      pendingRequests.delete(correlationId);
      reject(new Error(`Request ${correlationId} timed out`));
    }, timeoutMs);

    pendingRequests.set(correlationId, { resolve, reject, timer });

    fetch("http://localhost:3500/v1.0/publish/pubsub/requests", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        correlationId,
        replyTopic: "replies",
        replyAppId: "requester-service",
        payload,
      }),
    });
  });
}

// Handle replies
app.post("/replies", (req, res) => {
  const { correlationId, result } = req.body.data;
  const pending = pendingRequests.get(correlationId);
  if (pending) {
    clearTimeout(pending.timer);
    pendingRequests.delete(correlationId);
    pending.resolve(result);
  }
  res.sendStatus(200);
});
```

## Responder Implementation

```javascript
app.post("/requests", async (req, res) => {
  const { correlationId, replyTopic, payload } = req.body.data;

  // Process the request
  const result = await processRequest(payload);

  // Publish the reply
  await fetch(`http://localhost:3500/v1.0/publish/pubsub/${replyTopic}`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ correlationId, result }),
  });

  res.sendStatus(200);
});
```

## Subscription Configuration

```javascript
app.get("/dapr/subscribe", (req, res) => {
  res.json([
    {
      pubsubname: "pubsub",
      topic: "requests",
      route: "/requests",
    },
    {
      pubsubname: "pubsub",
      topic: "replies",
      route: "/replies",
    },
  ]);
});
```

## Using Reply-To Topic per Requester

For isolation, each requester service can use a unique reply topic:

```javascript
const replyTopic = `replies-${process.env.APP_ID}`;
```

This prevents one service from accidentally consuming another service's replies.

## Trade-offs vs Dapr Service Invocation

| Aspect | Service Invocation | Pub/Sub Request-Reply |
|---|---|---|
| Latency | Low (synchronous) | Higher (async round-trip) |
| Resilience | Fails if responder is down | Queues until responder recovers |
| Complexity | Low | Higher (correlation management) |
| Load smoothing | None | Yes - broker buffers spikes |

## Summary

Request-reply over Dapr pub/sub uses correlation IDs and reply topics to achieve bidirectional async communication. The requester publishes a request with a `correlationId`, the responder publishes the result back to the reply topic, and the requester matches it by ID. This pattern trades latency for resilience when the responder may be slow or temporarily unavailable.
