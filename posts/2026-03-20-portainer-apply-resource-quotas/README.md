# How to Apply Resource Quotas in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Resource Quota, Namespace, DevOps

Description: Learn how to configure resource quotas in Portainer's Kubernetes environments to prevent individual teams or applications from consuming excessive cluster resources.

## Introduction

Resource quotas in Kubernetes limit the total compute resources (CPU, memory) and object counts (pods, services, PVCs) that can be consumed within a namespace. Portainer provides a UI to configure these quotas, making it easier for administrators to enforce fair resource sharing across teams and environments.

## Prerequisites

- Portainer CE or BE with a Kubernetes environment
- Admin or namespace-admin access
- Kubernetes cluster with multiple namespaces

## Understanding Resource Quotas

A ResourceQuota object in Kubernetes defines limits for:

- **Compute resources**: CPU requests/limits, memory requests/limits
- **Object counts**: Max pods, services, PVCs, configmaps, secrets
- **Storage**: Total PVC capacity, storage class limits
- **Priority classes**: Resource limits per priority class

## Step 1: Enable Resource Quotas in Portainer

### Via Portainer UI

1. Log into Portainer as admin.
2. Select your Kubernetes environment.
3. Go to **Namespaces**.
4. Click on the target namespace or create a new one.
5. In the namespace settings, find the **Resource quota** section.
6. Enable **Resource quota** toggle.
7. Configure:
   - **CPU**: Total CPU cores limit (e.g., `4` cores)
   - **Memory**: Total memory limit (e.g., `8 Gi`)
8. Save.

Portainer creates the ResourceQuota object automatically.

## Step 2: Create Resource Quotas via KubeShell

For more granular control, use `kubectl` in the KubeShell:

```yaml
# resource-quota-team.yaml - Team namespace quota

apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: backend-team
spec:
  hard:
    # Compute resources
    requests.cpu: "4"          # Max total CPU requests
    requests.memory: 8Gi       # Max total memory requests
    limits.cpu: "8"            # Max total CPU limits
    limits.memory: 16Gi        # Max total memory limits

    # Object counts
    pods: "20"                 # Max number of pods
    services: "10"             # Max number of services
    persistentvolumeclaims: "5" # Max number of PVCs
    secrets: "20"              # Max number of secrets
    configmaps: "20"           # Max number of configmaps

    # Storage
    requests.storage: "100Gi"  # Max total storage requests
```

```bash
# Apply in KubeShell
kubectl apply -f resource-quota-team.yaml
```

## Step 3: Set Default Resource Requests with LimitRange

Resource quotas require containers to specify resource requests. Use LimitRange to set defaults:

```yaml
# limitrange-defaults.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: backend-team
spec:
  limits:
    - type: Container
      default:                  # Default limits if not specified
        cpu: "500m"
        memory: 256Mi
      defaultRequest:           # Default requests if not specified
        cpu: "100m"
        memory: 128Mi
      max:                      # Maximum per-container limits
        cpu: "2"
        memory: 2Gi
      min:                      # Minimum per-container requests
        cpu: "50m"
        memory: 64Mi
    - type: PersistentVolumeClaim
      max:
        storage: 20Gi           # Max PVC size
```

```bash
kubectl apply -f limitrange-defaults.yaml -n backend-team
```

## Step 4: Configure Quotas via the Portainer API

```bash
TOKEN="your-portainer-token"
ENDPOINT_ID=1
NAMESPACE="backend-team"

# Apply ResourceQuota via Kubernetes API through Portainer
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/endpoints/${ENDPOINT_ID}/kubernetes/api/v1/namespaces/${NAMESPACE}/resourcequotas" \
  -d '{
    "apiVersion": "v1",
    "kind": "ResourceQuota",
    "metadata": {
      "name": "team-quota",
      "namespace": "backend-team"
    },
    "spec": {
      "hard": {
        "requests.cpu": "4",
        "requests.memory": "8Gi",
        "limits.cpu": "8",
        "limits.memory": "16Gi",
        "pods": "20"
      }
    }
  }' | jq .
```

## Step 5: Check Quota Usage

```bash
# Check quota usage in a namespace
kubectl describe quota -n backend-team

# Example output:
# Name:            team-quota
# Namespace:       backend-team
# Resource                Used    Hard
# --------                ----    ----
# limits.cpu              2       8
# limits.memory           4Gi     16Gi
# pods                    8       20
# requests.cpu            1       4
# requests.memory         2Gi     8Gi

# Check via Portainer API
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/${ENDPOINT_ID}/kubernetes/api/v1/namespaces/${NAMESPACE}/resourcequotas" | \
  jq .
```

## Step 6: Quota Enforcement in Action

When a deployment exceeds the quota:

```bash
# Attempt to deploy more pods than allowed
kubectl run test-pod-$i --image=nginx -n backend-team

# Error when quota exceeded:
# Error from server (Forbidden): pods "test-pod-21" is forbidden:
# exceeded quota: team-quota, requested: pods=1, used: pods=20, limited: pods=20
```

## Step 7: Namespace Templates with Quotas

Create a script to provision new team namespaces with standard quotas:

```bash
#!/bin/bash
# create-team-namespace.sh - Provision a new team namespace with quotas

TEAM_NAME=$1
CPU_LIMIT="${2:-4}"
MEMORY_LIMIT="${3:-8Gi}"

echo "Creating namespace for team: $TEAM_NAME"

kubectl create namespace "$TEAM_NAME"

cat << EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: $TEAM_NAME
spec:
  hard:
    requests.cpu: "${CPU_LIMIT}"
    requests.memory: "${MEMORY_LIMIT}"
    limits.cpu: "$((CPU_LIMIT * 2))"
    limits.memory: "$((${MEMORY_LIMIT%Gi} * 2))Gi"
    pods: "30"
    services: "15"
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: $TEAM_NAME
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
EOF

echo "Namespace $TEAM_NAME created with quotas: CPU=$CPU_LIMIT, Memory=$MEMORY_LIMIT"
```

## Conclusion

Resource quotas in Portainer's Kubernetes environments ensure fair resource sharing between teams and prevent runaway workloads from consuming all cluster resources. Configure quotas at the namespace level through Portainer's UI or kubectl, use LimitRange to enforce per-container defaults, and script namespace provisioning to ensure every new team gets appropriate resource boundaries from day one.
