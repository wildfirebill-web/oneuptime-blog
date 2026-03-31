# How to Handle Service Invocation Errors Gracefully in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Error Handling, Service Invocation, Resiliency, Fault Tolerance

Description: Learn how to handle Dapr service invocation errors gracefully using status code mapping, fallback patterns, dead-letter queuing, and structured error responses.

---

## Dapr Service Invocation Error Codes

When a Dapr service invocation fails, the sidecar returns a standard HTTP error code:

| Status Code | Cause |
|-------------|-------|
| 400 | Bad request from the application |
| 403 | Access control policy denied |
| 404 | App ID or method not found |
| 408 | Request timed out |
| 500 | Application error |
| 503 | Service unavailable or circuit open |

## Basic Error Handling in Node.js

```javascript
async function getOrder(orderId) {
  try {
    const res = await axios.get(
      `http://localhost:3500/v1.0/invoke/order-service/method/orders/${orderId}`
    );
    return res.data;
  } catch (err) {
    const status = err.response?.status;

    if (status === 404) {
      throw new NotFoundError(`Order ${orderId} not found`);
    }
    if (status === 408) {
      throw new TimeoutError('Order service timed out');
    }
    if (status === 503) {
      // Circuit is open - use cached data
      return await orderCache.get(orderId);
    }
    throw new ServiceError(`Order service error: ${status}`);
  }
}
```

## Structured Error Responses from Services

Make your services return structured error bodies:

```javascript
// order-service
app.get('/orders/:id', async (req, res) => {
  const order = await db.findOrder(req.params.id);
  if (!order) {
    return res.status(404).json({
      code: 'ORDER_NOT_FOUND',
      message: `Order ${req.params.id} does not exist`,
      orderId: req.params.id,
    });
  }
  res.json(order);
});
```

```javascript
// Caller parses the structured error
} catch (err) {
  if (err.response?.status === 404) {
    const { code, message, orderId } = err.response.data;
    console.error(`${code}: ${message} [orderId=${orderId}]`);
  }
}
```

## Fallback Pattern

Provide a fallback response when the service is unavailable:

```javascript
async function getProductPrice(sku) {
  try {
    const res = await axios.get(
      `http://localhost:3500/v1.0/invoke/pricing-service/method/prices/${sku}`
    );
    return res.data.price;
  } catch {
    // Fall back to cached price
    const cached = await redisClient.get(`price:${sku}`);
    if (cached) return parseFloat(cached);
    // Last resort - use default price
    return 0;
  }
}
```

## Dead-Letter Queue for Failed Requests

For critical operations, queue failed requests for reprocessing:

```javascript
async function chargePayment(payload) {
  try {
    return await axios.post(
      'http://localhost:3500/v1.0/invoke/payment-service/method/charge',
      payload
    );
  } catch (err) {
    if (err.response?.status >= 500) {
      // Publish to dead-letter topic for retry
      await daprClient.pubsub.publish('pubsub', 'payment-dlq', {
        payload,
        error: err.message,
        timestamp: new Date().toISOString(),
        retryCount: (payload.retryCount || 0) + 1,
      });
    }
    throw err;
  }
}
```

## Configuring Resiliency with Error Matching

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: payment-resiliency
spec:
  policies:
    retries:
      retryOn5xx:
        policy: exponential
        maxRetries: 3
        duration: 1s
        matching:
          httpStatusCodes: "500,502,503,504"
  targets:
    apps:
      payment-service:
        retry: retryOn5xx
```

## Summary

Handle Dapr service invocation errors by mapping HTTP status codes to typed application exceptions, returning structured error bodies from services, and implementing fallback logic for 503 circuit-open responses. Use dead-letter queuing for critical operations that must eventually succeed. Configure Resiliency policies to automatically retry on 5xx errors.
