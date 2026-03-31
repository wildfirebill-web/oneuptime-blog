# How to Use Dynamic Configuration in Dapr Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Dynamic Configuration, Runtime Configuration, Feature Flag, Microservice

Description: Learn how to implement dynamic configuration in Dapr applications so runtime settings change without service restarts using the Configuration API and subscriptions.

---

Dynamic configuration lets your application change behavior at runtime without redeployment. Dapr's Configuration API with subscription support is the building block that makes this possible. This guide shows practical patterns for implementing dynamic config in production microservices.

## What Makes Configuration "Dynamic"

Configuration is dynamic when:
1. It can be changed without restarting the service
2. The change takes effect within seconds, not minutes
3. The service reacts appropriately to the new value

The Dapr Configuration API with Redis supports all three requirements out of the box.

## Pattern 1: Dynamic Rate Limits

Rate limits often need adjustment without a full deployment. Store them in Dapr config:

```bash
redis-cli SET myapp||rate-limit-rps "500"
redis-cli SET myapp||rate-limit-burst "1000"
```

In the application, watch for changes:

```javascript
const { DaprClient } = require('@dapr/dapr');
const Bottleneck = require('bottleneck');

let rateLimiter = new Bottleneck({ minTime: 2 }); // 500 rps

const dapr = new DaprClient();

async function initDynamicRateLimit() {
  await dapr.configuration.subscribeWithKeys(
    'appconfig',
    ['rate-limit-rps'],
    async (config) => {
      const rps = parseInt(config.items['rate-limit-rps'].value);
      const minTime = Math.floor(1000 / rps);
      rateLimiter = new Bottleneck({ minTime });
      console.log(`Rate limit updated to ${rps} RPS`);
    }
  );
}
```

## Pattern 2: Dynamic Log Level

Change log verbosity without redeployment - especially useful during incidents:

```python
import logging
import httpx

logger = logging.getLogger(__name__)
current_log_level = logging.INFO

async def watch_log_level():
    async with httpx.AsyncClient(timeout=None) as client:
        async with client.stream(
            "GET",
            "http://localhost:3500/v1.0-alpha1/configuration/appconfig/subscribe",
            params={"key": "log-level"}
        ) as response:
            async for line in response.aiter_lines():
                if not line.startswith("data:"):
                    continue
                import json
                event = json.loads(line[5:])
                new_level = event["items"].get("log-level", {}).get("value")
                if new_level:
                    level = getattr(logging, new_level.upper(), logging.INFO)
                    logging.getLogger().setLevel(level)
                    logger.info(f"Log level changed to {new_level}")
```

Trigger the change:

```bash
redis-cli SET myapp||log-level "debug"
# All running instances pick up the new level within seconds
```

## Pattern 3: Dynamic Connection Pool Size

Adjust database connection pool size in response to load:

```go
var poolSize atomic.Int32

func init() {
    poolSize.Store(10) // default
}

func startConfigWatcher(client dapr.Client) {
    ctx := context.Background()
    client.SubscribeConfigurationItems(
        ctx,
        "appconfig",
        []string{"db-pool-size"},
        func(id string, items map[string]*dapr.ConfigurationItem) {
            if item, ok := items["db-pool-size"]; ok {
                newSize, err := strconv.Atoi(item.Value)
                if err == nil && newSize > 0 && newSize <= 100 {
                    poolSize.Store(int32(newSize))
                    log.Printf("DB pool size updated to %d", newSize)
                    // Trigger pool resize
                    resizeConnectionPool(newSize)
                }
            }
        },
    )
}
```

## Pattern 4: Gradual Configuration Rollout

For risky configuration changes, use a canary value that applies only to a subset of instances:

```bash
# Apply to 10% of instances - combine with your deployment rollout
redis-cli SET myapp||experimental-cache-size "2048"
redis-cli SET myapp||experimental-rollout-percent "10"
```

```python
import random

def should_use_experimental():
    rollout_pct = int(config_cache.get("experimental-rollout-percent", "0"))
    return random.randint(1, 100) <= rollout_pct
```

## Summary

Dynamic configuration with Dapr enables real-time behavior changes across your microservices fleet without restarts or redeployments. The combination of the Configuration API for reads and subscription callbacks for push notifications makes patterns like dynamic rate limits, log level adjustments, and connection pool resizing straightforward to implement and operate safely.
