# How to Handle Binding Errors and Retries in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Binding, Error Handling, Retry, Resiliency

Description: Learn how to configure retry policies, circuit breakers, and error handling for Dapr bindings to build resilient, fault-tolerant microservices.

---

## Why Error Handling Matters for Dapr Bindings

Bindings connect your services to external systems - message queues, storage, APIs - that can fail, throttle, or become temporarily unavailable. Without proper error handling and retry configuration, a transient failure in a downstream system can cascade into data loss or service downtime.

Dapr provides built-in resiliency policies that you can attach to bindings to handle these scenarios automatically.

## Configuring Resiliency Policies

Dapr's Resiliency API lets you define retry, timeout, and circuit breaker policies and attach them to components.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: binding-resiliency
  namespace: default
spec:
  policies:
    retries:
      binding-retry-policy:
        policy: exponential
        initialInterval: 500ms
        maxInterval: 30s
        maxRetries: 5

    timeouts:
      binding-timeout: 10s

    circuitBreakers:
      binding-cb:
        maxRequests: 1
        interval: 30s
        timeout: 60s
        trip: consecutiveFailures >= 3

  targets:
    components:
      my-output-binding:
        outbound:
          retry: binding-retry-policy
          timeout: binding-timeout
          circuitBreaker: binding-cb
```

## Handling Input Binding Errors

For input bindings, return an appropriate HTTP status code from your handler:
- `200 OK` - job processed successfully
- `404 Not Found` or `5xx` - Dapr will retry delivery based on the binding's configured retry behavior

```javascript
const express = require("express");
const app = express();
app.use(express.json());

app.post("/my-input-binding", async (req, res) => {
  const message = req.body;

  try {
    await processMessage(message);
    res.status(200).send("OK");
  } catch (err) {
    if (err.retryable) {
      // Tell Dapr to retry this message
      console.error("Transient error, will retry:", err.message);
      res.status(500).send(err.message);
    } else {
      // Permanent failure - ack and move to dead letter if configured
      console.error("Permanent error, dropping message:", err.message);
      res.status(200).send("DROP");
    }
  }
});
```

## Handling Output Binding Errors

```javascript
const { DaprClient } = require("@dapr/dapr");
const client = new DaprClient();

async function sendWithFallback(data) {
  try {
    await client.binding.send("primary-binding", "create", data);
  } catch (err) {
    console.error("Primary binding failed:", err.message);

    // Fallback: write to a local queue or dead-letter store
    try {
      await client.binding.send("fallback-binding", "create", {
        originalBinding: "primary-binding",
        payload: data,
        error: err.message,
        timestamp: new Date().toISOString(),
      });
    } catch (fallbackErr) {
      console.error("Fallback also failed:", fallbackErr.message);
      throw fallbackErr;
    }
  }
}
```

## Resiliency Policy for Input Binding Triggers

For input bindings you also configure resiliency on the inbound path:

```yaml
  targets:
    components:
      my-input-binding:
        inbound:
          retry: binding-retry-policy
          timeout: binding-timeout
```

## Dead Letter Queues

Some bindings support dead letter queue configuration natively. For example, with AWS SQS:

```yaml
spec:
  type: bindings.aws.sqs
  version: v1
  metadata:
    - name: queueName
      value: "orders"
    - name: deadLetterQueueName
      value: "orders-dlq"
    - name: maxReceiveCount
      value: "3"
```

## Monitoring Failures

Use Dapr's metrics endpoint to observe binding failures:

```bash
curl http://localhost:9090/metrics | grep dapr_component_
```

Key metrics to watch:
- `dapr_component_output_binding_success_total`
- `dapr_component_output_binding_failure_total`
- `dapr_component_input_binding_success_total`

## Summary

Dapr's Resiliency API gives you declarative control over retry, timeout, and circuit breaker behavior for all bindings. By combining resiliency policies with proper application-level error handling and dead letter queue configuration, you can build binding integrations that are resilient to transient failures and gracefully handle permanent errors without losing data.
