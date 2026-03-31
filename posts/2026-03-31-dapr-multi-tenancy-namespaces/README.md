# How to Implement Multi-Tenancy with Dapr Namespaces

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Multi-Tenancy, Namespace, Kubernetes, Security

Description: Implement Dapr multi-tenancy using Kubernetes namespaces to isolate tenant workloads, components, and communication with scoping and mTLS trust domains.

---

## Multi-Tenancy Patterns with Dapr

Dapr supports multi-tenancy at the namespace level using Kubernetes namespaces as isolation boundaries. Each tenant gets their own namespace with dedicated Dapr components, ensuring one tenant's data and traffic cannot leak into another's namespace.

## Setting Up Tenant Namespaces

Create dedicated namespaces for each tenant:

```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b

# Label namespaces for Dapr injection
kubectl label namespace tenant-a dapr-enabled=true
kubectl label namespace tenant-b dapr-enabled=true
```

## Deploying Tenant-Specific Components

Each tenant namespace gets its own state store component pointing to isolated backend resources:

```yaml
# tenant-a-statestore.yaml
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
    value: "redis-tenant-a.tenant-a.svc.cluster.local:6379"
  - name: keyPrefix
    value: "tenant-a"
```

```yaml
# tenant-b-statestore.yaml
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
    value: "redis-tenant-b.tenant-b.svc.cluster.local:6379"
  - name: keyPrefix
    value: "tenant-b"
```

Both tenants use the same component name (`statestore`) but the namespace ensures they get different backends.

## Namespace-Scoped Service Invocation

By default, Dapr service invocation is namespace-scoped. An app in `tenant-a` can only invoke services in the same namespace without explicit cross-namespace configuration:

```javascript
// This call from tenant-a can only reach tenant-a services
const response = await fetch(
  'http://localhost:3500/v1.0/invoke/orders-api/method/orders'
);
```

To call across namespaces (only do this for shared services, not tenant services):

```javascript
// Cross-namespace invocation - use with care in multi-tenant setups
const response = await fetch(
  'http://localhost:3500/v1.0/invoke/orders-api.shared-services/method/orders'
);
```

## Configure Namespace-Level Trust Domains

For stronger isolation, configure separate mTLS trust domains per tenant namespace:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: tenant-a-config
  namespace: tenant-a
spec:
  mtls:
    enabled: true
    workloadCertTTL: "24h"
    allowedClockSkew: "15m"
```

## Apply Network Policies

Enforce namespace isolation at the network level:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-a-isolation
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: tenant-a
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: tenant-a
```

## Summary

Dapr multi-tenancy using namespaces provides isolation through namespace-scoped components, service invocation boundaries, and network policies. Each tenant namespace gets dedicated component instances pointing to isolated backends, and network policies prevent cross-tenant communication. This model scales well when tenants are large enough to warrant dedicated infrastructure per namespace.
