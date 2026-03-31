# How to Use the Dapr Configuration API to Get Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Configuration API, Application Configuration, Microservice

Description: Learn how to use the Dapr Configuration API to retrieve application configuration values from a centralized store using HTTP and gRPC calls.

---

The Dapr Configuration API provides a building block for reading application configuration from a centralized store. Unlike the Secrets API which is optimized for sensitive data, the Configuration API is designed for runtime settings like feature flags, thresholds, and environment-specific values that may change over time.

## Configuration API vs Secrets API

Understanding the difference helps you use the right tool:

- **Configuration API**: For non-sensitive settings that may change at runtime. Supports subscription to changes.
- **Secrets API**: For sensitive credentials. Read-only, no subscription support.

Use the Configuration API for things like rate limits, feature flags, A/B test parameters, and service discovery settings.

## Set Up Redis as Configuration Store

Redis is the most commonly used configuration store with Dapr. Deploy it first:

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
EOF
```

Define the Dapr configuration store component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: appconfig
  namespace: production
spec:
  type: configuration.redis
  version: v1
  metadata:
    - name: redisHost
      value: "redis.production.svc.cluster.local:6379"
    - name: redisPassword
      secretKeyRef:
        name: redis-secret
        key: password
```

## Add Configuration Values

Use the Redis CLI to add configuration values:

```bash
# Connect to Redis
kubectl exec -it deployment/redis -- redis-cli

# Add configuration items
SET myapp||max-retries "3"
SET myapp||timeout-seconds "30"
SET myapp||log-level "info"
SET myapp||feature-new-ui "false"
```

The key format is `<app-id>||<key-name>`.

## Retrieve Configuration via HTTP

```bash
# Get a single key
curl "http://localhost:3500/v1.0-alpha1/configuration/appconfig?key=max-retries"
```

Response:

```json
{
  "items": {
    "max-retries": {
      "value": "3",
      "version": "",
      "metadata": {}
    }
  }
}
```

Retrieve multiple keys at once:

```bash
curl "http://localhost:3500/v1.0-alpha1/configuration/appconfig?key=max-retries&key=timeout-seconds&key=log-level"
```

## Retrieve Configuration in Node.js

```javascript
const { DaprClient } = require('@dapr/dapr');

const client = new DaprClient();

async function getAppConfig() {
  const config = await client.configuration.get(
    'appconfig',
    ['max-retries', 'timeout-seconds', 'log-level']
  );

  return {
    maxRetries: parseInt(config.items['max-retries'].value),
    timeoutSeconds: parseInt(config.items['timeout-seconds'].value),
    logLevel: config.items['log-level'].value
  };
}

const appConfig = await getAppConfig();
console.log(`Max retries: ${appConfig.maxRetries}`);
```

## Using Configuration in Your Application Logic

```python
import httpx
from functools import lru_cache

@lru_cache(maxsize=1)
def get_config_cached():
    # Cache the config for 60 seconds
    resp = httpx.get(
        "http://localhost:3500/v1.0-alpha1/configuration/appconfig",
        params={"key": ["max-retries", "timeout-seconds"]}
    )
    return resp.json()["items"]

def get_max_retries() -> int:
    return int(get_config_cached()["max-retries"]["value"])
```

## Summary

The Dapr Configuration API provides a standardized way to retrieve application settings from a centralized store backed by Redis, PostgreSQL, or Azure App Configuration. It is distinct from the Secrets API and is intended for non-sensitive runtime configuration that your services need to read at startup or during operation, with the added benefit of subscription support for detecting changes.
