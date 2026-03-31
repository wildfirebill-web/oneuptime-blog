# How to Implement Tenant-Specific Configuration with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Configuration, Multi-Tenant, SaaS, Isolation

Description: Store and retrieve per-tenant configuration using the Dapr Configuration API, with tenant-scoped keys and live subscription for real-time config updates.

---

## Tenant Configuration Challenges

SaaS applications need to serve different behavior to different tenants: different feature sets, limits, themes, and integrations. Dapr's Configuration API provides a natural key-prefix model for tenant-scoped settings.

## Key Naming Convention

Use `tenant:{tenantId}:{setting}` as the key format to namespace each tenant's config:

```bash
# Tenant A: enterprise tier
redis-cli MSET \
  "tenant-config||tenant:acme:maxUsers" "1000" \
  "tenant-config||tenant:acme:features" '["analytics","exports","sso"]' \
  "tenant-config||tenant:acme:theme" "dark" \
  "tenant-config||tenant:acme:rateLimitRps" "500"

# Tenant B: starter tier
redis-cli MSET \
  "tenant-config||tenant:startup-xyz:maxUsers" "10" \
  "tenant-config||tenant:startup-xyz:features" '["basic"]' \
  "tenant-config||tenant:startup-xyz:theme" "light" \
  "tenant-config||tenant:startup-xyz:rateLimitRps" "20"
```

## Dapr Configuration Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: tenant-config
spec:
  type: configuration.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis:6379"
```

## Tenant Config Service

```python
import json
from functools import lru_cache
from dapr.clients import DaprClient
import threading

class TenantConfigService:
    def __init__(self, store: str = "tenant-config"):
        self.store = store
        self._cache: dict[str, dict[str, str]] = {}
        self._lock = threading.RLock()

    def get_config(self, tenant_id: str) -> dict[str, str]:
        with self._lock:
            if tenant_id in self._cache:
                return self._cache[tenant_id]
        return self._load_tenant(tenant_id)

    def _load_tenant(self, tenant_id: str) -> dict[str, str]:
        keys = [
            f"tenant:{tenant_id}:maxUsers",
            f"tenant:{tenant_id}:features",
            f"tenant:{tenant_id}:theme",
            f"tenant:{tenant_id}:rateLimitRps",
        ]
        with DaprClient() as client:
            result = client.get_configuration(self.store, keys)
            cfg = {
                k.split(":")[-1]: v.value
                for k, v in result.configuration.items()
            }
        with self._lock:
            self._cache[tenant_id] = cfg
        return cfg

    def has_feature(self, tenant_id: str, feature: str) -> bool:
        cfg = self.get_config(tenant_id)
        features = json.loads(cfg.get("features", "[]"))
        return feature in features

    def get_rate_limit(self, tenant_id: str) -> int:
        cfg = self.get_config(tenant_id)
        return int(cfg.get("rateLimitRps", "10"))

    def watch_tenant(self, tenant_id: str):
        keys = [
            f"tenant:{tenant_id}:maxUsers",
            f"tenant:{tenant_id}:features",
            f"tenant:{tenant_id}:rateLimitRps",
        ]
        def _run():
            with DaprClient() as client:
                sub = client.subscribe_configuration(self.store, keys)
                for resp in sub:
                    for key, item in resp.items():
                        field = key.split(":")[-1]
                        with self._lock:
                            if tenant_id in self._cache:
                                self._cache[tenant_id][field] = item.value
                        print(f"Tenant {tenant_id} config updated: {field}={item.value}")

        threading.Thread(target=_run, daemon=True).start()
```

## FastAPI Integration

```python
from fastapi import FastAPI, Request, HTTPException
from tenant_config import TenantConfigService

app = FastAPI()
tenant_svc = TenantConfigService()

@app.middleware("http")
async def tenant_middleware(request: Request, call_next):
    tenant_id = request.headers.get("X-Tenant-ID")
    if not tenant_id:
        return HTTPException(400, "Missing X-Tenant-ID header")
    request.state.tenant_id = tenant_id
    return await call_next(request)

@app.get("/api/analytics")
async def get_analytics(request: Request):
    tenant_id = request.state.tenant_id
    if not tenant_svc.has_feature(tenant_id, "analytics"):
        raise HTTPException(403, "Analytics feature not available on your plan")
    return {"data": await fetch_analytics(tenant_id)}
```

## Updating a Tenant's Config

```bash
# Upgrade tenant to enterprise tier
redis-cli SET "tenant-config||tenant:startup-xyz:features" '["basic","analytics","exports"]'
redis-cli SET "tenant-config||tenant:startup-xyz:rateLimitRps" "200"
```

## Summary

Dapr's Configuration API provides a flexible key-prefix model for tenant-scoped configuration in SaaS applications. Tenant configs load on first access, are cached in memory, and can be watched for live updates. This approach avoids per-tenant databases for configuration while still giving complete isolation between tenant settings.
