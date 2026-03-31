# How to Use Namespaced Actors in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Namespace, Kubernetes, Multi-Tenancy

Description: Learn how to use Dapr namespaced actors to isolate actor instances by Kubernetes namespace, enabling multi-tenant deployments with separate state and routing.

---

Namespaced actors in Dapr allow you to run the same actor type in multiple Kubernetes namespaces, each with fully isolated state and placement. This is essential for multi-tenant SaaS architectures where different tenants must not share actor state.

## How Namespaced Actors Work

By default, Dapr's placement service treats actor types globally. With namespace support enabled, the placement service partitions actor placement by namespace, so `Counter/001` in `tenant-a` is distinct from `Counter/001` in `tenant-b`.

## Enabling Namespace-Scoped Actor Placement

In the Dapr Helm chart values, enable namespace-scoped placement:

```yaml
# values.yaml for dapr helm chart
dapr_placement:
  namespace_scoped: true
```

Or apply the annotation to your namespace:

```bash
kubectl label namespace tenant-a dapr.io/enable-api-logging=true
```

## Deploying Actor Services Per Namespace

Deploy the same actor service image in each namespace with namespace-specific component configurations:

```yaml
# tenant-a/statestore.yaml
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
    value: "redis-tenant-a:6379"
  - name: actorStateStore
    value: "true"
```

```yaml
# tenant-b/statestore.yaml
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
    value: "redis-tenant-b:6379"
  - name: actorStateStore
    value: "true"
```

## Calling Actors Across Namespaces

Actors in different namespaces cannot call each other directly. Cross-namespace communication should go through service invocation with namespace-qualified app IDs:

```bash
# Invoke actor in tenant-a from a service in another namespace
curl -X POST http://localhost:3500/v1.0/actors/Counter/counter-001/method/Increment \
  -H "Dapr-Namespace: tenant-a" \
  -H "Content-Type: application/json" \
  -d '{"amount": 1}'
```

## Self-Hosted Namespaces

In self-hosted mode, namespaces are simulated using the `NAMESPACE` environment variable:

```bash
NAMESPACE=tenant-a dapr run --app-id counter-service \
  --app-port 8080 \
  -- ./counter-service
```

## Verifying Namespace Isolation

Check that actors are registered in the correct namespace via the placement HTTP endpoint:

```bash
curl http://localhost:9090/placement/state
```

The response includes actor type registrations with their namespace tags.

## Best Practices

- Use separate Redis instances per namespace for hard state isolation between tenants.
- Never share the `statestore` component across namespaces in multi-tenant deployments.
- Apply Kubernetes NetworkPolicy to prevent cross-namespace Dapr sidecar communication.
- Monitor per-namespace actor counts using Dapr's Prometheus metrics endpoint.

## Summary

Namespaced actors in Dapr enable clean multi-tenant isolation by partitioning actor placement and state by Kubernetes namespace. This pattern is well-suited for SaaS platforms where tenant data must remain strictly separated. Configuring namespace-scoped placement and dedicated state stores per namespace provides both operational and security isolation.
