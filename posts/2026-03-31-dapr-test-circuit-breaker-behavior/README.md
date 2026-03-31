# How to Test Circuit Breaker Behavior in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Circuit Breaker, Resiliency, Testing, Microservice

Description: Learn how to test circuit breaker behavior in Dapr by simulating failures, verifying state transitions, and validating recovery using resiliency policies.

---

## Overview

Circuit breakers in Dapr protect your microservices from cascading failures by automatically stopping calls to unhealthy services. Testing circuit breaker behavior ensures your resiliency policies behave as expected under real failure conditions.

Dapr's circuit breaker follows three states: closed (normal operation), open (all calls fail fast), and half-open (probe calls to check recovery). Testing each state transition is essential for building reliable systems.

## Setting Up a Circuit Breaker Resiliency Policy

Define a resiliency policy with a circuit breaker in your component YAML:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: test-circuit-breaker
spec:
  policies:
    circuitBreakers:
      targetCB:
        maxRequests: 1
        interval: 8s
        timeout: 30s
        trip: consecutiveFailures >= 3
  targets:
    apps:
      order-service:
        circuitBreaker: targetCB
```

The `trip` expression defines when the circuit opens. Here, three consecutive failures trigger it.

## Simulating Failures with a Fault Service

Create a simple HTTP service that returns errors on demand to simulate the downstream failure:

```javascript
const express = require('express');
const app = express();
let failCount = 0;
let maxFails = 5;

app.get('/order', (req, res) => {
  if (failCount < maxFails) {
    failCount++;
    return res.status(500).json({ error: 'Simulated failure' });
  }
  res.json({ orderId: '123', status: 'processed' });
});

app.post('/reset', (req, res) => {
  failCount = 0;
  res.json({ message: 'Reset successful' });
});

app.listen(3001, () => console.log('Fault service running on port 3001'));
```

## Testing the Open State

Use a test script to confirm the circuit opens after consecutive failures:

```bash
# Send multiple requests to trigger the circuit breaker
for i in $(seq 1 5); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    http://localhost:3500/v1.0/invoke/order-service/method/order
done
```

After three failures you should see Dapr returning a `500` with a circuit breaker error message. Subsequent calls return immediately without hitting the downstream service.

## Verifying the Half-Open Probe

After the `timeout` period (30 seconds in this example), Dapr moves the circuit to half-open and allows one probe request. Confirm by checking Dapr sidecar logs:

```bash
kubectl logs -l app=your-app -c daprd | grep "circuit breaker"
```

Example log output:

```
time="2026-03-31T10:00:00Z" level=info msg="circuit breaker 'targetCB' changed state: half-open"
time="2026-03-31T10:00:31Z" level=info msg="circuit breaker 'targetCB' changed state: closed"
```

## Automating Circuit Breaker Tests

Integrate into a test suite using the Dapr HTTP API:

```bash
#!/bin/bash
set -e

# Step 1: Trigger failures
for i in $(seq 1 3); do
  curl -s http://localhost:3500/v1.0/invoke/order-service/method/order
done

# Step 2: Verify circuit is open
STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:3500/v1.0/invoke/order-service/method/order)

if [ "$STATUS" == "500" ]; then
  echo "PASS: Circuit breaker opened as expected"
else
  echo "FAIL: Expected 500, got $STATUS"
  exit 1
fi

# Step 3: Wait for timeout and verify recovery
sleep 35
STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:3500/v1.0/invoke/order-service/method/order)

echo "Recovery status: $STATUS"
```

## Summary

Testing Dapr circuit breakers requires simulating downstream failures, verifying state transitions from closed to open to half-open, and confirming recovery behavior. By combining fault injection services, Dapr sidecar logs, and automated scripts, you can build confident regression tests for your resiliency policies. Always test circuit breaker configuration changes in a staging environment before promoting to production.
