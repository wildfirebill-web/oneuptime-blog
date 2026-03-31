# How to Configure Dapr for High-Latency Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Network, Latency, Timeout, Configuration

Description: Tune Dapr timeouts, retries, and resiliency policies to handle high-latency networks gracefully without cascading failures or premature timeouts.

---

## The Impact of High Latency on Dapr

In high-latency environments - cross-region deployments, satellite connections, or congested networks - Dapr's default timeouts can cause premature failures. Default service invocation timeouts and pub/sub retry intervals are tuned for low-latency LAN environments. Proper tuning prevents unnecessary failures while avoiding indefinite waits.

## Understanding Dapr Timeout Defaults

Default timeouts in Dapr:
- Service invocation: no default timeout (relies on HTTP client defaults)
- Actor invocation: 60 seconds
- State store operations: component-specific (typically 5-30 seconds)
- Pub/Sub ack timeout: component-specific

## Configuring Resiliency Policies

Dapr's Resiliency API (1.7+) lets you declare timeout, retry, and circuit breaker policies:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: high-latency-policy
spec:
  policies:
    timeouts:
      defaultTimeout: 30s
      longTimeout: 120s

    retries:
      defaultRetryPolicy:
        policy: exponential
        maxInterval: 60s
        maxRetries: 5
        duration: 5s

    circuitBreakers:
      defaultCircuitBreaker:
        maxRequests: 1
        interval: 10s
        timeout: 30s
        trip: consecutiveFailures >= 3

  targets:
    apps:
      checkout:
        timeout: longTimeout
        retry: defaultRetryPolicy
        circuitBreaker: defaultCircuitBreaker

    components:
      statestore:
        outbound:
          timeout: longTimeout
          retry: defaultRetryPolicy
```

## Per-Service Timeout Configuration

Apply resiliency policies to specific service invocations:

```yaml
  targets:
    apps:
      remote-service:
        timeout: 45s      # Extended for cross-region calls
        retry:
          policy: exponential
          maxInterval: 30s
          maxRetries: 3
          duration: 2s
      local-service:
        timeout: 5s       # Short timeout for same-region
```

## State Store Timeout Tuning

Configure state store component with extended timeouts:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: redis.remote-region:6379
    - name: redisConnectTimeout
      value: "10000"    # 10 seconds connection timeout (ms)
    - name: redisReadTimeout
      value: "30000"    # 30 seconds read timeout (ms)
    - name: redisWriteTimeout
      value: "30000"    # 30 seconds write timeout (ms)
    - name: poolSize
      value: "20"
    - name: poolTimeout
      value: "30000"
```

## Pub/Sub Configuration for High Latency

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
    - name: host
      value: amqp://remote-rabbitmq:5672
    - name: connectionAttempts
      value: "10"
    - name: connectionWait
      value: "5s"
    - name: heartBeat
      value: "60s"    # Extended heartbeat for high-latency links
    - name: publishTimeout
      value: "30s"
```

## Actor Configuration for High Latency

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: actor-high-latency-config
spec:
  actor:
    actorIdleTimeout: 60m
    actorScanInterval: 60s      # Less frequent for high-latency
    drainOngoingCallTimeout: 120s  # Extended drain timeout
    drainRebalancedActors: true
```

## Application-Level Timeout Handling

```javascript
const DAPR_HTTP_PORT = process.env.DAPR_HTTP_PORT || 3500;

async function invokeWithRetry(appId, method, data, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), 30000); // 30s

      const response = await fetch(
        `http://localhost:${DAPR_HTTP_PORT}/v1.0/invoke/${appId}/method/${method}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(data),
          signal: controller.signal,
        }
      );
      clearTimeout(timeoutId);
      return await response.json();
    } catch (err) {
      if (attempt === maxRetries - 1) throw err;
      await new Promise(r => setTimeout(r, Math.pow(2, attempt) * 1000));
    }
  }
}
```

## Summary

Configuring Dapr for high-latency networks centers on Dapr's Resiliency API, which provides declarative timeout, retry, and circuit breaker policies at the service and component level. Extended connection timeouts in component configurations and application-level retry logic with exponential backoff prevent cascading failures when network round-trips take significantly longer than LAN-optimized defaults expect.
