# How to Organize Dapr Component Files in a Repository

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Components, Repository Structure, Best Practices, DevOps

Description: Learn how to organize Dapr component YAML files in a repository using a clear directory structure, environment separation, and shared component patterns.

---

## The Challenge of Managing Dapr Components

As your Dapr application grows, you accumulate many component files: state stores, pub/sub brokers, secret stores, bindings, middleware, and configuration. Without a clear structure, these files become hard to maintain, especially when you need different configurations per environment (dev, staging, production).

## Recommended Directory Structure

```text
/dapr
  /components
    /base               # Shared base configurations
      statestore.yaml
      pubsub.yaml
      secrets.yaml
    /dev                # Development overrides
      statestore.yaml   # Points to local Redis
      pubsub.yaml       # Points to local RabbitMQ
    /staging
      statestore.yaml   # Points to staging Redis cluster
      pubsub.yaml
    /production
      statestore.yaml   # Points to prod Redis with TLS
      pubsub.yaml
  /config
    tracing.yaml        # Dapr configuration (tracing, middleware)
    resiliency.yaml     # Retry and circuit breaker policies
  /subscriptions
    order-events.yaml
    payment-events.yaml
```

## Base Component Files

Define base components with placeholder values or environment-agnostic settings:

```yaml
# dapr/components/base/statestore.yaml
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
    value: "redis:6379"
  - name: redisPassword
    secretKeyRef:
      name: redis-secret
      key: password
  - name: actorStateStore
    value: "true"
  - name: keyPrefix
    value: "none"
```

```yaml
# dapr/components/base/pubsub.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: messagebus
  namespace: default
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
  - name: host
    value: "amqp://rabbitmq:5672"
  - name: durable
    value: "true"
  - name: deletedWhenUnused
    value: "false"
```

## Environment-Specific Overrides

Override specific fields for each environment:

```yaml
# dapr/components/dev/statestore.yaml
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
    value: "localhost:6379"
  - name: redisPassword
    value: ""
  - name: actorStateStore
    value: "true"
```

```yaml
# dapr/components/production/statestore.yaml
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
    value: "redis-cluster.production.svc.cluster.local:6379"
  - name: redisPassword
    secretKeyRef:
      name: redis-secret
      key: password
  - name: enableTLS
    value: "true"
  - name: actorStateStore
    value: "true"
```

## Using Kustomize for Environment Layering

Kustomize integrates well with Dapr components for environment management:

```yaml
# dapr/components/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - statestore.yaml
  - pubsub.yaml
  - secrets.yaml
```

```yaml
# dapr/components/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base
patches:
  - path: statestore.yaml
  - path: pubsub.yaml
```

Apply for production:

```bash
kubectl apply -k dapr/components/production
```

## Component Naming Conventions

Use consistent naming conventions across all components:

```text
Component Type     Name Convention         Example
-------------      ---------------         -------
State store        statestore              statestore
Pub/sub            messagebus              messagebus
Secret store       {provider}-secrets      vault-secrets, k8s-secrets
Binding (input)    {source}-input          kafka-input, cron-input
Binding (output)   {target}-output         email-output, s3-output
Middleware         {function}-middleware   ratelimit-middleware
```

## Scoping Components to Specific Apps

Use scopes to restrict which apps can access each component:

```yaml
# dapr/components/base/payment-secrets.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: payment-secrets
  namespace: default
spec:
  type: secretstores.aws.secretmanager
  version: v1
  metadata:
  - name: region
    value: "us-east-1"
scopes:
  - payment-service
  - billing-service
```

## Makefile for Component Management

Create a Makefile to simplify deploying components:

```makefile
# Makefile
ENV ?= dev

.PHONY: apply-components
apply-components:
	kubectl apply -k dapr/components/$(ENV)
	@echo "Applied Dapr components for environment: $(ENV)"

.PHONY: validate-components
validate-components:
	@for f in dapr/components/$(ENV)/*.yaml; do \
		kubectl apply --dry-run=client -f $$f && echo "OK: $$f" || echo "FAIL: $$f"; \
	done

.PHONY: run-local
run-local:
	dapr run --app-id my-service \
	         --resources-path dapr/components/dev \
	         --config dapr/config/tracing.yaml \
	         -- go run main.go
```

Usage:

```bash
# Deploy to staging
make apply-components ENV=staging

# Validate production components
make validate-components ENV=production

# Run locally with dev components
make run-local
```

## GitOps Integration

For GitOps workflows, organize components alongside application manifests:

```text
/k8s
  /apps
    /payment-service
      deployment.yaml
      service.yaml
  /dapr
    /components
      statestore.yaml
      pubsub.yaml
    /config
      tracing.yaml
  /overlays
    /dev
    /staging
    /production
```

## Summary

A well-organized Dapr component repository separates base configurations from environment-specific overrides, uses consistent naming conventions, and applies scopes to enforce least-privilege access. Using Kustomize for layering and a Makefile for deployment commands reduces errors and makes it straightforward to promote configurations across environments. Keeping component files alongside Kubernetes manifests in the same repository enables unified GitOps workflows.
