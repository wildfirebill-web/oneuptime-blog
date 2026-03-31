# How to Implement Environment-Specific Configs with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Configuration, Environment, DevOps, Backend

Description: Manage development, staging, and production configs in Redis using namespaced hash keys, inheritance overrides, and promotion workflows between environments.

---

Managing config across dev, staging, and prod environments is a common pain point. Redis hashes with environment namespacing let you define base configs once and override only what differs per environment.

## Namespaced Config Keys

Prefix every config key with the environment name:

```bash
# Base / default values
HSET config:default:payments timeout_ms 5000
HSET config:default:payments max_retries 3

# Production overrides
HSET config:production:payments timeout_ms 3000
HSET config:production:payments max_retries 5

# Staging inherits from default but has a test gateway
HSET config:staging:payments gateway_url "https://sandbox.payments.io"
```

## Config Resolution with Inheritance

Resolve the effective config by merging default values with environment-specific overrides:

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def get_config(service: str, env: str) -> dict:
    base = r.hgetall(f"config:default:{service}")
    overrides = r.hgetall(f"config:{env}:{service}")
    # Override base with env-specific values
    return {**base, **overrides}
```

## Promoting Configs Between Environments

Copy staging config to production after validation:

```python
def promote_config(service: str, from_env: str, to_env: str):
    source_key = f"config:{from_env}:{service}"
    target_key = f"config:{to_env}:{service}"
    values = r.hgetall(source_key)
    if not values:
        raise ValueError(f"No config found for {service} in {from_env}")
    pipe = r.pipeline()
    pipe.delete(target_key)
    for k, v in values.items():
        pipe.hset(target_key, k, v)
    pipe.execute()
```

## Diff Configs Between Environments

Quickly identify what differs between staging and production:

```python
def diff_configs(service: str, env_a: str, env_b: str) -> dict:
    cfg_a = get_config(service, env_a)
    cfg_b = get_config(service, env_b)
    diff = {}
    all_keys = set(cfg_a) | set(cfg_b)
    for key in all_keys:
        a, b = cfg_a.get(key), cfg_b.get(key)
        if a != b:
            diff[key] = {"a": a, "b": b}
    return diff
```

## Listing All Services in an Environment

```bash
# Use SCAN to list all config keys for production
SCAN 0 MATCH "config:production:*" COUNT 100
```

## Restricting Write Access per Environment

Use Redis ACL to allow write access to production only from authorized deployment pipelines:

```bash
ACL SETUSER deploy_prod on >strongpassword ~config:production:* +HSET +HDEL
ACL SETUSER app_user on >apppassword ~config:* +HGETALL +HGET
```

## Summary

Environment-namespaced Redis hashes let you maintain one canonical base config and layer environment-specific overrides on top. Merge resolution at read time keeps storage minimal. Promotion workflows, diff utilities, and ACL-based access control make this approach safe for production use across multiple environments.

