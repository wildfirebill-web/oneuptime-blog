# How to Subscribe to Configuration Changes in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Configuration API, Subscription, Real-Time Configuration, Microservice

Description: Learn how to subscribe to configuration changes in Dapr so your application reacts instantly when feature flags or runtime settings are updated.

---

One of the most powerful features of the Dapr Configuration API is the ability to subscribe to configuration changes in real time. Instead of polling for changes or restarting services after a config update, your application receives a push notification whenever a watched key changes.

## How Configuration Subscriptions Work

Dapr uses SSE (Server-Sent Events) over HTTP to stream configuration change notifications to your application. Your service subscribes to one or more keys and receives events whenever those keys are updated in the backing store.

## Setting Up a Subscription via HTTP

Open a subscription stream:

```bash
curl -N "http://localhost:3500/v1.0-alpha1/configuration/appconfig/subscribe?key=feature-new-ui&key=log-level"
```

The response is a stream of JSON events:

```json
{"items":{"feature-new-ui":{"value":"true","version":"2","metadata":{}}}}
```

## Subscribing in Node.js

```javascript
const { DaprClient } = require('@dapr/dapr');

const client = new DaprClient();

async function watchConfig() {
  // Start subscription and get subscription ID
  const subscriptionId = await client.configuration.subscribeWithKeys(
    'appconfig',
    ['feature-new-ui', 'log-level', 'max-retries'],
    async (config) => {
      console.log('Configuration changed:', config.items);

      // React to specific changes
      if ('feature-new-ui' in config.items) {
        const enabled = config.items['feature-new-ui'].value === 'true';
        updateFeatureFlag('new-ui', enabled);
      }

      if ('log-level' in config.items) {
        setLogLevel(config.items['log-level'].value);
      }
    }
  );

  console.log(`Subscribed with ID: ${subscriptionId}`);
  return subscriptionId;
}

// Unsubscribe when done
async function stopWatching(subscriptionId) {
  await client.configuration.unsubscribe('appconfig', subscriptionId);
}
```

## Subscribing in Python

```python
import asyncio
import httpx

async def watch_config():
    config_state = {
        "feature-new-ui": False,
        "log-level": "info"
    }

    async with httpx.AsyncClient(timeout=None) as client:
        async with client.stream(
            "GET",
            "http://localhost:3500/v1.0-alpha1/configuration/appconfig/subscribe",
            params={"key": ["feature-new-ui", "log-level"]}
        ) as response:
            async for line in response.aiter_lines():
                if line.startswith("data:"):
                    import json
                    event = json.loads(line[5:])
                    for key, item in event.get("items", {}).items():
                        config_state[key] = item["value"]
                        print(f"Config updated: {key} = {item['value']}")

asyncio.run(watch_config())
```

## Reacting to Changes in a Long-Running Service

The pattern for a long-running service is to hold the current config in memory and update it when events arrive:

```go
package main

import (
    "context"
    "log"
    dapr "github.com/dapr/go-sdk/client"
)

var currentConfig = map[string]string{
    "max-retries":    "3",
    "timeout-seconds": "30",
}

func main() {
    client, err := dapr.NewClient()
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    ctx := context.Background()
    subscriptionID, err := client.SubscribeConfigurationItems(
        ctx,
        "appconfig",
        []string{"max-retries", "timeout-seconds"},
        func(id string, items map[string]*dapr.ConfigurationItem) {
            for key, item := range items {
                log.Printf("Config change: %s = %s", key, item.Value)
                currentConfig[key] = item.Value
            }
        },
    )
    if err != nil {
        log.Fatal(err)
    }
    defer client.UnsubscribeConfigurationItems(ctx, "appconfig", subscriptionID)

    // Run service...
    select {}
}
```

## Summary

Dapr Configuration subscriptions enable real-time config updates without service restarts. By subscribing to specific keys in your configuration store, your application receives push notifications whenever values change and can immediately update in-memory state, toggle feature flags, or adjust runtime parameters - making dynamic configuration a first-class feature of your microservices architecture.
