# How to Use Kustomize for Dapr Application Deployment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kustomize, Deployment, Kubernetes, DevOps

Description: Learn how to use Kustomize to manage Dapr application deployments with base configurations and environment-specific overlays for components and annotations.

---

Kustomize is built into `kubectl` and lets you customize Kubernetes manifests without templating. For Dapr applications, Kustomize manages deployment annotations, component configurations, and environment-specific overrides through a base-overlay pattern.

## Base Configuration

Define the common Dapr deployment in the base:

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "8080"
        dapr.io/config: "dapr-config"
        dapr.io/log-level: "info"
    spec:
      containers:
      - name: order-service
        image: order-service:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

```yaml
# base/components/pubsub.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis:6379"
  - name: redisPassword
    secretKeyRef:
      name: redis-credentials
      key: password
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
- components/pubsub.yaml
- components/state-store.yaml
- config/dapr-config.yaml
```

## Production Overlay

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
- ../../base
images:
- name: order-service
  newName: myrepo/order-service
  newTag: "2.1.0"
patches:
- path: replica-patch.yaml
  target:
    kind: Deployment
    name: order-service
- path: redis-host-patch.yaml
  target:
    kind: Component
    name: pubsub
- path: log-level-patch.yaml
  target:
    kind: Deployment
    name: order-service
```

```yaml
# overlays/production/replica-patch.yaml
- op: replace
  path: /spec/replicas
  value: 3
```

```yaml
# overlays/production/redis-host-patch.yaml
- op: replace
  path: /spec/metadata/0/value
  value: "prod-redis.production.svc.cluster.local:6379"
```

```yaml
# overlays/production/log-level-patch.yaml
- op: replace
  path: /spec/template/metadata/annotations/dapr.io~1log-level
  value: "warn"
```

## Strategic Merge Patches

Use strategic merge patches to add Dapr resource limits:

```yaml
# overlays/production/resource-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/sidecar-cpu-request: "100m"
        dapr.io/sidecar-memory-request: "128Mi"
        dapr.io/sidecar-cpu-limit: "500m"
        dapr.io/sidecar-memory-limit: "512Mi"
```

## Applying Kustomize Overlays

```bash
# Preview the production manifests
kubectl kustomize overlays/production/

# Apply production overlay
kubectl apply -k overlays/production/

# Diff against running cluster state
kubectl diff -k overlays/production/

# Apply development overlay
kubectl apply -k overlays/development/ -n development

# Update image tag using Kustomize edit
cd overlays/production
kustomize edit set image order-service=myrepo/order-service:2.2.0
kubectl apply -k .
```

## Summary

Kustomize manages Dapr application deployments through a base-overlay pattern where the base contains common Dapr annotations and component definitions, and overlays patch environment-specific values like replica counts, Redis hosts, and log levels. Use `kubectl diff -k` to preview changes before applying, and `kustomize edit set image` to update image tags without editing YAML manually.
