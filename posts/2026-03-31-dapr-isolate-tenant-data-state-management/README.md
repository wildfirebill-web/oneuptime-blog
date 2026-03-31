# How to Isolate Tenant Data with Dapr State Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, State Management, Multi-Tenancy, Security, Isolation

Description: Isolate tenant data in Dapr state management using key prefixing, separate state store backends, and namespace-scoped components to prevent data leakage.

---

## Why Tenant Data Isolation Matters

In a multi-tenant system, tenant A must never read or write tenant B's state. Dapr's state management provides several mechanisms to enforce this isolation, ranging from key prefixing within a shared store to completely separate state backends per tenant.

## Option 1 - App ID Key Prefixing (Basic Isolation)

Dapr automatically prefixes all state keys with the app ID by default. This provides logical separation within a shared store:

```bash
# App with app-id "tenant-a-service" stores key "profile"
# Actual Redis key: "tenant-a-service||profile"

# App with app-id "tenant-b-service" stores the same key "profile"
# Actual Redis key: "tenant-b-service||profile"
```

Control the prefix strategy in the component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: my-namespace
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis:6379
  - name: keyPrefix
    value: "appid"
```

**Warning:** This only provides logical separation - both tenants share the same Redis instance. A misconfigured key prefix can still expose cross-tenant data.

## Option 2 - Separate State Store Backends

For strong isolation, each tenant uses a dedicated state store component pointing to a separate database:

```yaml
# Tenant A gets its own Redis instance
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: tenant-a
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis.tenant-a.svc.cluster.local:6379"
scopes:
- tenant-a-api
```

```yaml
# Tenant B gets its own Redis instance
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: tenant-b
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis.tenant-b.svc.cluster.local:6379"
scopes:
- tenant-b-api
```

Both components have the same name `statestore` but live in different namespaces, so each tenant's application automatically connects to the correct one.

## Option 3 - Tenant ID in Application Code

When sharing a backend, enforce tenant isolation in application code using the tenant ID as a key prefix:

```javascript
const daprClient = new DaprClient();

async function getTenantState(tenantId, key) {
  // Always prefix with tenant ID
  const tenantKey = `${tenantId}:${key}`;
  return await daprClient.state.get('statestore', tenantKey);
}

async function setTenantState(tenantId, key, value) {
  const tenantKey = `${tenantId}:${key}`;
  await daprClient.state.save('statestore', [{
    key: tenantKey,
    value: value
  }]);
}
```

Extract `tenantId` from a JWT claim or a request header validated at the ingress layer.

## Encrypting State Per Tenant

For compliance requirements, encrypt state with tenant-specific keys:

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
    value: redis:6379
  - name: primaryEncryptionKey
    secretKeyRef:
      name: tenant-encryption-keys
      key: tenant-a-key
```

## Summary

Dapr tenant data isolation ranges from automatic app ID key prefixing to fully separate state store backends per namespace. For strong isolation and compliance requirements, deploy separate backend instances per tenant with namespace-scoped Dapr components. When sharing backends, always enforce tenant-prefixed keys in application code and validate the tenant identity at the ingress layer before any state operation.
