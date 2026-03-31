# How to Use Dapr with Score Workload Specification

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Score, Platform Engineering, Workload, Specification, Portability

Description: Use Score workload specification with Dapr to define application resource needs in a platform-agnostic format and generate Dapr component configurations for different environments.

---

## What Is Score?

Score is an open-source workload specification format that describes application resource requirements in a developer-centric, platform-agnostic way. Instead of writing Kubernetes YAML directly, developers declare what their service needs (e.g., a database, a message queue) and Score translates that into the target platform's format.

Score integrates with Dapr by generating Dapr component YAML from high-level resource declarations.

## Installing Score CLI

```bash
# Install score-k8s
brew install score-spec/tap/score-k8s

# Verify
score-k8s --version
```

## Writing a Score Workload File

The `score.yaml` file describes what a service needs without specifying how:

```yaml
apiVersion: score.dev/v1b1
metadata:
  name: order-processor

containers:
  order-processor:
    image: myregistry/order-processor:latest
    variables:
      DAPR_HTTP_PORT: "3500"
      ORDER_TOPIC: "${resources.order-queue.topic}"
    resources:
      limits:
        memory: "256Mi"
        cpu: "250m"

service:
  ports:
    http:
      port: 8080
      targetPort: 8080

resources:
  order-queue:
    type: queue             # Platform-agnostic resource type
    params:
      topic: orders

  session-store:
    type: redis             # Will become a Dapr state store
    params:
      ttl: 3600
```

## Score Extensions for Dapr

Create Score extension files that map resource types to Dapr components:

```yaml
# score-dapr-extensions.yaml
extensions:
  queue:
    dapr:
      component_type: pubsub.kafka
      version: v1
      metadata:
        - name: brokers
          value: kafka:9092
        - name: consumerGroup
          value: "${workload}-consumers"

  redis:
    dapr:
      component_type: state.redis
      version: v1
      metadata:
        - name: redisHost
          value: redis:6379
        - name: ttlInSeconds
          value: "${params.ttl}"
```

## Generating Kubernetes + Dapr Manifests

Run score-k8s to generate Kubernetes and Dapr manifests:

```bash
score-k8s generate score.yaml \
  --extensions score-dapr-extensions.yaml \
  --output ./k8s/

ls k8s/
# deployment.yaml
# service.yaml
# dapr-statestore.yaml
# dapr-pubsub.yaml
```

The generated Dapr component YAML:

```yaml
# k8s/dapr-statestore.yaml (generated)
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: session-store
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: redis:6379
    - name: ttlInSeconds
      value: "3600"
scopes:
  - order-processor
```

## Multi-Environment Workflow

Use Score overrides for environment-specific values:

```bash
# Development
score-k8s generate score.yaml \
  --override-file overrides/dev.yaml \
  --output ./k8s/dev/

# Production
score-k8s generate score.yaml \
  --override-file overrides/production.yaml \
  --output ./k8s/production/
```

Production override file:

```yaml
# overrides/production.yaml
containers:
  order-processor:
    resources:
      limits:
        memory: "512Mi"
        cpu: "500m"

resources:
  order-queue:
    params:
      brokers: kafka.production.svc:9092
  session-store:
    params:
      ttl: 86400
      enableTLS: true
```

## CI/CD Integration

Integrate Score generation into your CI pipeline:

```yaml
# .github/workflows/deploy.yaml
jobs:
  generate-and-deploy:
    steps:
      - name: Generate manifests
        run: |
          score-k8s generate score.yaml \
            --extensions score-dapr-extensions.yaml \
            --override-file overrides/${{ env.ENVIRONMENT }}.yaml \
            --output ./k8s/

      - name: Apply to cluster
        run: kubectl apply -f ./k8s/ -n ${{ env.ENVIRONMENT }}
```

## Summary

Score workload specification provides a developer-friendly abstraction above Kubernetes and Dapr YAML. Developers declare resource needs (queues, caches) without knowing Dapr component types, and Score extensions map those resources to the appropriate Dapr component configuration. This separation of concerns lets platform teams control Dapr component standards while application developers focus on what their service needs rather than how it is configured.
