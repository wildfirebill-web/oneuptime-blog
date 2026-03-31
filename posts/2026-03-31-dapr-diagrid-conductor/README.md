# How to Use Dapr with Diagrid Conductor

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Diagrid, Conductor, Management, Platform

Description: Use Diagrid Conductor to manage, monitor, and operate Dapr deployments across multiple Kubernetes clusters with a centralized control plane.

---

## Overview

Diagrid Conductor is a managed Dapr control plane from the creators of Dapr at Diagrid. It provides a centralized dashboard, fleet management, component configuration, and operational tools for Dapr deployments running across one or more Kubernetes clusters.

## Prerequisites

- Diagrid account (sign up at diagrid.dev)
- `diagrid` CLI installed
- One or more Kubernetes clusters
- Dapr installed or installable on target clusters

## Installing the Diagrid CLI

```bash
# macOS/Linux
curl -o- https://downloads.diagrid.io/install.sh | bash

# Verify installation
diagrid version
```

## Connecting a Cluster to Conductor

```bash
# Login to Diagrid
diagrid login

# Create a project
diagrid project create my-platform

# Connect a Kubernetes cluster
diagrid cluster connect my-cluster \
  --project my-platform \
  --kubeconfig ~/.kube/config
```

Conductor installs a lightweight agent in the cluster:

```bash
# Verify the agent is running
kubectl get pods -n diagrid-system
```

## Deploying Dapr Apps via Conductor

Define applications using Conductor's CRDs:

```yaml
apiVersion: core.diagrid.io/v1
kind: AppID
metadata:
  name: order-service
  namespace: default
spec:
  appPort: 8080
  config:
    logLevel: info
    tracing:
      samplingRate: "1"
  scopes:
  - pubsub:orders
  - statestore:order-db
```

Apply and monitor via CLI:

```bash
diagrid appid apply -f order-service.yaml
diagrid appid list
diagrid appid get order-service
```

## Managing Components via Conductor

Create Dapr components centrally:

```bash
# Create a Redis state store via CLI
diagrid component create statestore \
  --type state.redis \
  --version v1 \
  --metadata "redisHost=redis:6379" \
  --metadata "redisPassword=secret"

# List all components across clusters
diagrid component list
```

## Monitoring with Conductor Dashboard

View real-time metrics in the Conductor UI:

```bash
# Open the Conductor dashboard
diagrid dashboard open

# Or view metrics from CLI
diagrid metrics get order-service \
  --metric dapr_http_server_request_count \
  --window 1h
```

## Conductor API for Automation

Integrate Conductor into CI/CD pipelines:

```bash
# Get Conductor API token
export DIAGRID_API_TOKEN=$(diagrid auth token)

# Deploy component via API
curl -X POST https://api.diagrid.io/v1/projects/my-platform/components \
  -H "Authorization: Bearer ${DIAGRID_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "pubsub",
    "type": "pubsub.redis",
    "version": "v1",
    "metadata": {"redisHost": "redis:6379"}
  }'
```

## Summary

Diagrid Conductor simplifies Dapr fleet management with centralized component configuration, real-time monitoring, and multi-cluster support. By moving Dapr operational concerns into Conductor, teams spend less time managing Dapr infrastructure and more time building business logic with Dapr's building blocks.
