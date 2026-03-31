# How to Migrate a Single Service to Dapr First

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Migration, Strategy, Service, Kubernetes

Description: Learn the step-by-step process to migrate a single existing service to Dapr without disrupting other services, using a strangler fig pattern approach.

---

## Why Start with a Single Service

Migrating all services to Dapr at once is high-risk. A single-service migration:
- Validates your component configurations in production
- Builds team confidence and knowledge
- Limits the blast radius if issues arise
- Creates a reference implementation for other teams

## Selecting the Right First Service

Choose a service that is:
- Self-contained with few direct dependencies on other internal services
- Not on the critical revenue path
- Owned by a team enthusiastic about Dapr
- Uses at least one resource Dapr can abstract (Redis, Kafka, secrets)

## Step 1: Audit the Service's External Dependencies

Document what the service currently connects to:

```bash
# Review current service dependencies
grep -r "redis\|kafka\|vault\|postgres" ./services/notification-service/
```

Map each dependency to a Dapr component:

```yaml
dependency_map:
  redis_client: state.redis
  kafka_producer: pubsub.kafka
  vault_client: secretstores.vault
```

## Step 2: Create Component YAML Files

Create Dapr component files that match existing infrastructure:

```yaml
# components/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: production
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: redis-master.production:6379
    - name: redisPassword
      secretKeyRef:
        name: redis-secret
        key: password
scopes:
  - notification-service  # Limit to this service only during migration
```

## Step 3: Update the Service Code

Replace the direct Redis client with the Dapr SDK:

```go
// Before: Direct Redis
rdb := redis.NewClient(&redis.Options{Addr: "redis:6379"})
err := rdb.Set(ctx, "key", value, 0).Err()

// After: Dapr SDK
client, _ := dapr.NewClient()
err := client.SaveState(ctx, "statestore", "key", value, nil)
```

## Step 4: Add Dapr Annotations to Kubernetes Deployment

```yaml
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "notification-service"
        dapr.io/app-port: "8080"
        dapr.io/log-level: "info"
        dapr.io/config: "tracing-config"
```

## Step 5: Deploy to Staging

```bash
kubectl apply -f components/ -n staging
kubectl apply -f deployment.yaml -n staging

# Verify sidecar is running
kubectl get pods -n staging -l app=notification-service
# Should show 2/2 containers (app + daprd)
```

## Step 6: Validate and Monitor

Run smoke tests and monitor for 48 hours before promoting to production:

```bash
# Check sidecar health
kubectl exec -it notification-service-xxx -c daprd -- \
  wget -qO- http://localhost:3500/v1.0/healthz

# Watch for errors
kubectl logs -l app=notification-service -c daprd --follow | grep -i error
```

## Step 7: Promote to Production with Feature Flag

Use a feature flag to route a percentage of traffic to the Dapr-enabled version:

```bash
# Deploy Dapr version alongside legacy
kubectl apply -f deployment-dapr.yaml -n production

# Gradually shift traffic using Ingress weight or service mesh
```

## Summary

Migrating a single service to Dapr first reduces risk by limiting the blast radius and creating a reference implementation for your organization. The process involves auditing dependencies, creating component YAML files scoped to only the target service, updating application code to use the Dapr SDK, and monitoring closely in staging before promoting to production.
