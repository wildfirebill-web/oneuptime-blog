# How to Configure Rancher Agent Resource Allocation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Agent, Resource Management, cattle-system

Description: Configure resource requests and limits for Rancher agents running in managed clusters to prevent resource contention and ensure reliable cluster management.

## Introduction

Every Rancher-managed cluster runs several cattle-system agents: cattle-cluster-agent, cattle-node-agent, and fleet-agent. These agents communicate with Rancher server and maintain cluster state. Without proper resource limits, these agents can consume excessive resources or be starved during high utilization. This guide covers configuring Rancher agent resources appropriately.

## Prerequisites

- Rancher-managed Kubernetes clusters
- Cluster admin access
- Understanding of your cluster's resource profile

## Step 1: Check Current Agent Resource Usage

```bash
# Check agent resource usage in a managed cluster
kubectl top pods -n cattle-system
kubectl top pods -n fleet-system

# Get current resource configuration
kubectl describe deployment cattle-cluster-agent -n cattle-system
kubectl describe deployment fleet-agent -n fleet-system

# Check if agents are hitting limits
kubectl get events -n cattle-system | grep -i "OOMKilled\|Evicted"
```

## Step 2: Configure cattle-cluster-agent Resources

```bash
# Patch cattle-cluster-agent resource limits
kubectl patch deployment cattle-cluster-agent \
  -n cattle-system \
  --type=json \
  -p='[
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/0/resources",
      "value": {
        "requests": {
          "cpu": "200m",
          "memory": "256Mi"
        },
        "limits": {
          "cpu": "1000m",
          "memory": "1Gi"
        }
      }
    }
  ]'

# For large clusters (1000+ nodes), increase limits
kubectl patch deployment cattle-cluster-agent \
  -n cattle-system \
  --type=json \
  -p='[
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/0/resources",
      "value": {
        "requests": {
          "cpu": "500m",
          "memory": "512Mi"
        },
        "limits": {
          "cpu": "2000m",
          "memory": "2Gi"
        }
      }
    }
  ]'
```

## Step 3: Configure Fleet Agent Resources

```bash
# Fleet agent handles GitOps deployments
kubectl patch deployment fleet-agent \
  -n cattle-fleet-system \
  --type=json \
  -p='[
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/0/resources",
      "value": {
        "requests": {
          "cpu": "100m",
          "memory": "128Mi"
        },
        "limits": {
          "cpu": "500m",
          "memory": "512Mi"
        }
      }
    }
  ]'
```

## Step 4: Configure via Rancher Helm Chart

```yaml
# rancher-values.yaml - Set agent resources at install time
# For the managed cluster's agent configuration, use Rancher API

# Check the agent configuration via Rancher UI:
# Cluster Edit > Advanced Options > Agent Resource Limits
```

```bash
# Configure agent resources via Rancher API
curl -X PUT \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "agentResources": {
      "requests": {
        "cpu": "200m",
        "memory": "256Mi"
      },
      "limits": {
        "cpu": "1000m",
        "memory": "1Gi"
      }
    }
  }' \
  "https://rancher.example.com/v3/clusters/c-xxxxx"
```

## Step 5: Node Selector for Agents

```bash
# Pin agents to specific nodes to avoid resource contention
# with application workloads

kubectl patch deployment cattle-cluster-agent \
  -n cattle-system \
  --type=json \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/nodeSelector",
      "value": {
        "node-role.kubernetes.io/control-plane": "true"
      }
    }
  ]'

# This ensures agents run on control plane nodes,
# leaving worker nodes for application workloads
```

## Step 6: Configure Priority Class

```yaml
# rancher-agent-priority.yaml - Ensure agents are high priority
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: rancher-agent-critical
value: 1000000
globalDefault: false
description: "Priority class for Rancher agents"
---
# Apply to cattle-cluster-agent
kubectl patch deployment cattle-cluster-agent \
  -n cattle-system \
  --type=json \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/priorityClassName",
      "value": "rancher-agent-critical"
    }
  ]'
```

## Step 7: Configure Toleration for Tainted Nodes

```bash
# If control plane nodes are tainted, add tolerations to agents
kubectl patch deployment cattle-cluster-agent \
  -n cattle-system \
  --type=json \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/tolerations",
      "value": [
        {
          "key": "node-role.kubernetes.io/control-plane",
          "effect": "NoSchedule",
          "operator": "Exists"
        },
        {
          "key": "node-role.kubernetes.io/master",
          "effect": "NoSchedule",
          "operator": "Exists"
        }
      ]
    }
  ]'
```

## Step 8: Monitor Agent Health

```bash
# Watch agent reconnection patterns
kubectl logs -n cattle-system \
  deployment/cattle-cluster-agent \
  --since=1h | grep -E "error|reconnect|disconnect"

# Check websocket connection to Rancher
kubectl exec -n cattle-system \
  $(kubectl get pod -n cattle-system -l app=cattle-cluster-agent -o name) \
  -- env | grep CATTLE_SERVER

# Alert if agent is frequently restarting
kubectl get pods -n cattle-system -w
```

## Conclusion

Properly configured Rancher agent resources are critical for maintaining reliable cluster management. Under-resourced agents experience disconnections from Rancher, which can cause management operations to fail. For clusters with heavy workloads, ensure agents are scheduled on dedicated nodes with appropriate resource limits. The priority class configuration ensures agents survive resource pressure that would evict normal workloads, maintaining the management plane's availability.
