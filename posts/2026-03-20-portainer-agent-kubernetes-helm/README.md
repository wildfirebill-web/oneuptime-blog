# How to Install Portainer Agent on Kubernetes via Helm - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Agent, Kubernetes, Helm, Installation

Description: Install the Portainer Agent in a Kubernetes cluster using Helm for centralized management from a Portainer server.

## Introduction

The Portainer Agent Helm chart provides the recommended way to deploy the Portainer Agent in Kubernetes. Helm handles RBAC, service accounts, and the DaemonSet configuration automatically.

## Prerequisites

- Kubernetes cluster running
- Helm 3 installed
- kubectl access to the cluster
- Portainer server running

## Step 1: Add the Portainer Helm Repository

```bash
helm repo add portainer https://portainer.github.io/k8s/
helm repo update

# Verify

helm search repo portainer
```

## Step 2: Install the Agent

```bash
# Basic installation
helm install portainer-agent \
  --create-namespace \
  -n portainer \
  portainer/portainer-agent

# With custom agent secret (must match Portainer server setting)
helm install portainer-agent \
  --create-namespace \
  -n portainer \
  portainer/portainer-agent \
  --set env.key=AGENT_SECRET \
  --set env.value="shared-agent-secret"
```

## Step 3: Expose the Agent

The agent runs as a DaemonSet and exposes port 9001. For Portainer to connect, expose the service:

```bash
# Check current service type
kubectl get svc portainer-agent -n portainer

# For external Portainer server, patch to LoadBalancer
kubectl patch svc portainer-agent -n portainer \
  -p '{"spec":{"type":"LoadBalancer"}}'

# Get external IP
kubectl get svc portainer-agent -n portainer -w
```

## Step 4: Add Kubernetes Environment to Portainer

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

AGENT_URL="tcp://AGENT_EXTERNAL_IP:9001"

curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints \
  -d "{
    \"name\": \"Production Kubernetes\",
    \"endpointCreationType\": 2,
    \"URL\": \"${AGENT_URL}\",
    \"type\": 7
  }"
```

## Helm Chart Configuration Options

```yaml
# values.yaml overrides
replicaCount: 1

image:
  repository: portainer/agent
  tag: latest
  pullPolicy: Always

service:
  type: NodePort    # or LoadBalancer, ClusterIP
  nodePort: 30778   # if NodePort

resources:
  limits:
    cpu: 100m
    memory: 256Mi

env:
  key: AGENT_SECRET
  value: "your-shared-secret"
```

```bash
helm install portainer-agent \
  -n portainer \
  portainer/portainer-agent \
  -f values.yaml
```

## Upgrading the Agent

```bash
# Update the Helm repo
helm repo update

# Upgrade to latest
helm upgrade portainer-agent \
  -n portainer \
  portainer/portainer-agent

# Check the upgrade
kubectl get pods -n portainer -w
```

## Conclusion

The Helm chart is the cleanest way to deploy the Portainer Agent in Kubernetes. It handles all the Kubernetes-specific configuration automatically. Once deployed, expose the agent service for external access if Portainer runs outside the cluster, or use ClusterIP if Portainer runs inside.
