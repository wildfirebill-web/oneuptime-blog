# How to Prefix State Keys by Application ID in Dapr

Author: [OneUptime](https://oneuptime.com)

Tags: Dapr, State Management, Key Prefix, Isolation, Microservice

Description: Learn how Dapr automatically prefixes state keys with the application ID to isolate state between services, and how to configure or override this behaviour.

---

## Introduction

When multiple Dapr applications share the same state store, key collisions are a real risk. A `user-42` key written by `orderservice` should not overwrite the same key written by `inventoryservice`. Dapr solves this by prepending the application ID to every key by default, creating automatic namespace isolation per service.

## Default Key Prefix Behaviour

```mermaid
graph TD
    A[orderservice writes key: order-001] -->|stored as: orderservice||order-001| Store[(State Store)]
    B[inventory writes key: order-001] -->|stored as: inventory||order-001| Store
    Store --> C[No collision - different entries]
```

The separator is `||` (double pipe). So:
- App ID: `orderservice`, key: `order-001` -> stored as `orderservice||order-001`
- App ID: `inventory`, key: `order-001` -> stored as `inventory||order-001`

## Configuring the Key Prefix

The `keyPrefix` metadata field on the state store component controls this behaviour:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: default
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: redis-master:6379
    - name: keyPrefix
      value: appid   # default
```

Available values:

| Value | Stored Key Format | Use Case |
|-------|------------------|----------|
| `appid` | `{appId}||{key}` | Per-app isolation (default) |
| `name` | `{componentName}||{key}` | Isolation by component name |
| `none` | `{key}` | Shared state, no prefix |

## Using the Default appid Prefix

No configuration change needed. Simply write and read state normally:

```bash
# App: orderservice
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[{"key": "order-001", "value": {"item": "laptop"}}]'

# Actual Redis key: orderservice||order-001
```

```bash
# Verify directly in Redis
redis-cli GET "orderservice||order-001"
# Output: {"item":"laptop"}
```

## Reading State with App ID Awareness

When you read state via Dapr, the sidecar automatically prepends your app ID, so you always use the short key:

```bash
# Reads orderservice||order-001 under the hood
curl http://localhost:3500/v1.0/state/statestore/order-001
```

## Cross-App Key Access

To read another app's state (for read-only scenarios), construct the full key with the prefix manually:

```bash
# inventory service reading orderservice's state
# URL-encode the || separator as %7C%7C
curl http://localhost:3500/v1.0/state/statestore/orderservice%7C%7Corder-001
```

This bypasses the normal app ID prefix logic. Only use this for read operations and document the cross-service dependency clearly.

## Switching to name Prefix

Use `name` prefix when you want isolation by component name rather than app ID. Useful when multiple apps share data within the same logical "namespace":

```yaml
spec:
  metadata:
    - name: keyPrefix
      value: name
```

With component name `statestore` and key `config-v1`, the stored key is `statestore||config-v1`.

## Switching to none Prefix

Disable the prefix entirely for fully shared state:

```yaml
spec:
  metadata:
    - name: keyPrefix
      value: none
```

```bash
# Both orderservice and inventory read/write the same "cart-42" entry
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -d '[{"key": "cart-42", "value": {"items": []}}]'
```

## Migrating Existing State to a Different Prefix

If you change `keyPrefix`, existing keys remain under the old prefix. Migrate them with a script:

```python
import redis

r = redis.Redis(host="localhost", port=6379)
old_prefix = "orderservice||"
new_prefix = ""  # switching to none

for old_key in r.scan_iter(f"{old_prefix}*"):
    new_key = old_key.decode().replace(old_prefix, new_prefix, 1)
    value = r.get(old_key)
    r.set(new_key, value)
    r.delete(old_key)
    print(f"Migrated {old_key} -> {new_key}")
```

## Verifying Key Prefixes

```bash
# List all keys for an app in Redis
redis-cli KEYS "orderservice||*"

# Count keys per app
redis-cli KEYS "orderservice||*" | wc -l
redis-cli KEYS "inventory||*" | wc -l

# Check what Dapr app ID is set
dapr list
```

## Summary

Dapr automatically prefixes state keys with the application ID using `{appId}||{key}` format, providing per-service state isolation out of the box. Control this behaviour with the `keyPrefix` metadata field on your state store component: use `appid` (default) for isolation, `name` for component-level namespacing, or `none` for fully shared state. When migrating between prefix modes, run a one-time key migration script to move existing data to the new key format.
