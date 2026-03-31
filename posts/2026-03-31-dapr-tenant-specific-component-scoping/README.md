# How to Implement Tenant-Specific Component Scoping in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Component, Scoping, Multi-Tenancy, Security

Description: Use Dapr component scoping to restrict which app IDs can access specific components, enabling fine-grained access control in multi-tenant Kubernetes deployments.

---

## What is Dapr Component Scoping?

Dapr component scoping allows you to specify exactly which application IDs can use a given component. Without scoping, any app in the same namespace can load and use any component. Scoping provides an additional authorization layer beyond namespace isolation.

## Basic Component Scoping

Add the `scopes` field to any component to restrict access:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: payments-store
  namespace: tenant-a
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis-payments:6379"
scopes:
- payments-api
- audit-service
```

Only apps with `dapr.io/app-id: "payments-api"` or `dapr.io/app-id: "audit-service"` in the `tenant-a` namespace will load this component. Other apps in the same namespace won't even see it.

## Scoping Pub/Sub Components

Separate publish and subscribe access for pub/sub components using the `publishingScopes` and `subscriptionScopes` fields:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: tenant-a
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis:6379
```

Use an allowed topics list in the subscription to control which apps subscribe to which topics:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: orders-sub
  namespace: tenant-a
spec:
  pubsubname: pubsub
  topic: orders
  routes:
    default: /orders/process
```

Combine with app-level scoping using the component's `scopes` field to limit which apps can publish.

## Denying Components to Specific Apps

You can explicitly deny access using a deny list pattern. If the `scopes` field is set and an app is NOT listed, it cannot use the component:

```yaml
# Only orders-api and billing-api can use this component
# All other apps in the namespace are denied
scopes:
- orders-api
- billing-api
```

## Auditing Component Access

Check which apps have access to which components:

```bash
# List all components and their scopes
kubectl get components -n tenant-a -o json | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
for item in data['items']:
    name = item['metadata']['name']
    scopes = item['spec'].get('scopes', ['ALL'])
    print(f'{name}: {scopes}')
"
```

## Combining Scoping with RBAC

Add Kubernetes RBAC to prevent unauthorized users from modifying component scopes:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: component-reader
  namespace: tenant-a
rules:
- apiGroups: ["dapr.io"]
  resources: ["components"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: component-manager
  namespace: tenant-a
rules:
- apiGroups: ["dapr.io"]
  resources: ["components"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
```

## Summary

Dapr component scoping provides fine-grained access control by restricting which app IDs can load and use specific components within a namespace. This is especially important in multi-tenant deployments where multiple applications share a namespace but must have limited access to sensitive components like secret stores and payment state stores. Combine scoping with RBAC to prevent unauthorized modifications to component access rules.
