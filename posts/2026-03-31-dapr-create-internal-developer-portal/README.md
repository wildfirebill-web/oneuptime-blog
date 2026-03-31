# How to Create an Internal Dapr Developer Portal

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Developer Portal, Platform Engineering, Backstage, Documentation

Description: Learn how to build an internal Dapr developer portal using Backstage or a wiki to centralize component catalogs, onboarding guides, and self-service tooling for your teams.

---

As Dapr adoption grows, engineers need a central place to discover available components, find onboarding guides, and self-service their way through common tasks. An internal developer portal reduces reliance on the platform team for routine questions.

## What an Internal Dapr Portal Provides

A good Dapr developer portal includes:

- Component catalog: what state stores, pub/sub, and secret stores are available and in which environments
- Getting started guides tailored to your organization's setup
- Runnable templates for new Dapr-enabled services
- Troubleshooting runbooks for common failures
- Links to observability dashboards for Dapr metrics

## Option 1: Backstage-Based Portal

Backstage is the most common choice for enterprise developer portals. Add Dapr components as catalog entities:

```yaml
# catalog/dapr-statestore-prod.yaml
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: statestore-prod
  description: "Redis-backed state store for production services"
  annotations:
    dapr.io/component-type: "state.redis"
    dapr.io/environment: "production"
  tags:
    - dapr
    - state-store
    - production
spec:
  type: dapr-component
  lifecycle: production
  owner: platform-team
```

Register the catalog file in `app-config.yaml`:

```yaml
catalog:
  locations:
    - type: file
      target: ./catalog/dapr-*.yaml
```

Teams can then search the Backstage catalog to discover available Dapr components before writing any code.

## Option 2: Wiki-Based Portal

For smaller teams, a Confluence or Notion wiki works well. Structure it as:

```
Dapr Internal Guide/
  Getting Started/
    Install Dapr locally
    Your first service
    Running in Kubernetes
  Component Catalog/
    State Stores (Redis - prod, staging; DynamoDB - prod)
    Pub/Sub (Kafka - prod; RabbitMQ - staging)
    Secret Stores (Vault - all environments)
  How-To Guides/
    Add state management to your service
    Subscribe to a topic
    Debug a failing Dapr call
  Troubleshooting/
    Common errors and fixes
    How to read Dapr logs
  Runbooks/
    Dapr sidecar not starting
    Message not being delivered
```

## Self-Service Templates

Provide scaffolding scripts that generate Dapr-ready service skeletons:

```bash
#!/bin/bash
# create-dapr-service.sh

SERVICE_NAME=$1
TEAM=$2
LANGUAGE=$3

echo "Creating Dapr service: ${TEAM}-${SERVICE_NAME}"

mkdir -p services/${SERVICE_NAME}
cat > services/${SERVICE_NAME}/k8s-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${SERVICE_NAME}
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "${TEAM}-${SERVICE_NAME}"
        dapr.io/app-port: "8080"
        dapr.io/config: "dapr-config"
EOF

echo "Service scaffold created at services/${SERVICE_NAME}/"
```

## Component Discovery API

Expose an API endpoint that lists available components per environment:

```bash
# Query the Dapr operator for deployed components
kubectl get components -A -o json | jq '[.items[] | {name: .metadata.name, type: .spec.type, namespace: .metadata.namespace}]'
```

Serve this as a JSON API from your portal backend so teams can query programmatically.

## Summary

An internal Dapr developer portal centralizes component discovery, onboarding guides, and self-service tooling. Use Backstage for enterprise-scale portals with component catalog integration, or a well-structured wiki for smaller teams. Self-service scaffolding scripts and runbooks reduce platform team toil and accelerate new service creation.
