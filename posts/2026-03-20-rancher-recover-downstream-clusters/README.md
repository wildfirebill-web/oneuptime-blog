# How to Recover Downstream Clusters After Rancher Failure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Disaster-recovery, Downstream-clusters, Kubernetes, Recovery

Description: Guide to recovering and reconnecting downstream Kubernetes clusters after a Rancher management server failure.

## Introduction

When Rancher fails, downstream clusters continue running their workloads independently. However, you lose the management plane-the ability to deploy, configure, and monitor through Rancher. This guide explains how to reconnect and recover downstream clusters after Rancher is restored.

## What Happens to Downstream Clusters During Rancher Failure

- Existing workloads continue running (pods, services, deployments)
- Persistent volumes remain attached and functional
- DNS and load balancers continue working
- Kubernetes API server is accessible directly via kubeconfig
- Rancher agents lose connection but don't crash workloads
- Fleet GitOps continues applying changes via local agent cache

## Step 1: Access Clusters Directly During Rancher Downtime

```bash
# If you have direct kubeconfig, use it to manage clusters

export KUBECONFIG=/path/to/cluster-kubeconfig.yaml

# Verify direct access
kubectl get nodes
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

# Emergency operations can be performed directly
kubectl scale deployment my-app --replicas=0 -n production
```

## Step 2: Verify Cluster State After Rancher Restore

After restoring Rancher, check cluster reconnection status:

```bash
# Check which clusters have reconnected
kubectl get clusters.management.cattle.io \
  -o custom-columns='NAME:.metadata.name,STATE:.status.conditions[?(@.type=="Ready")].status'

# Detailed status for a specific cluster
kubectl describe cluster my-cluster-name
```

## Step 3: Force Agent Reconnection

If a downstream cluster does not automatically reconnect:

```bash
# On the downstream cluster, restart the Rancher agent
kubectl rollout restart deployment cattle-cluster-agent \
  -n cattle-system

# Check agent logs
kubectl logs -n cattle-system \
  -l app=cattle-cluster-agent \
  --tail=50

# If agent is not running, redeploy using cluster import YAML
# Get import command from Rancher UI or API:
curl -s https://rancher.example.com/v3/clusters/c-xxxxx?action=generateKubeconfig \
  -H "Authorization: Bearer your-api-token" | jq -r '.config'
```

## Step 4: Re-Import Clusters That Cannot Auto-Reconnect

For clusters that fail to automatically reconnect:

```bash
# 1. Get the cluster import YAML from restored Rancher
# In Rancher UI: Cluster Management > Import Existing

# 2. Apply to the downstream cluster
kubectl apply -f https://rancher.example.com/v3/import/cluster-token.yaml

# 3. If SSL verification fails (new Rancher cert):
curl --insecure -sfL https://rancher.example.com/v3/import/cluster-token.yaml | kubectl apply -f -
```

## Step 5: Update Agent Server URL

If Rancher moved to a new IP or hostname:

```yaml
# Update cattle-cluster-agent deployment on downstream cluster
kubectl edit deployment cattle-cluster-agent -n cattle-system

# Or use kubectl patch:
kubectl patch deployment cattle-cluster-agent \
  -n cattle-system \
  --type='json' \
  -p='[{
    "op": "replace",
    "path": "/spec/template/spec/containers/0/env/0/value",
    "value": "https://new-rancher.example.com"
  }]'
```

## Step 6: Handle Certificate Changes

If Rancher's TLS certificate changed after recovery:

```bash
# Update the cluster agent's CA cert
kubectl -n cattle-system \
  delete secret cattle-credentials-xxx 2>/dev/null || true

# Re-register with new certificate
kubectl apply -f https://rancher.example.com/v3/import/new-cluster-token.yaml
```

## Step 7: Verify Full Functionality

```bash
#!/bin/bash
# verify-cluster-reconnection.sh

CLUSTER_NAME="$1"
echo "=== Verifying cluster: $CLUSTER_NAME ==="

# Check cluster status in Rancher
kubectl get clusters.management.cattle.io "$CLUSTER_NAME" \
  -o jsonpath='{.status.conditions}' | jq '.'

# Check agent pods on downstream cluster
KUBECONFIG="/path/to/${CLUSTER_NAME}-kubeconfig.yaml"
kubectl --kubeconfig="$KUBECONFIG" get pods -n cattle-system

# Test Rancher can reach the cluster
kubectl get clusterconnections.management.cattle.io \
  -n "$CLUSTER_NAME" 2>/dev/null

echo "Verification complete"
```

## Step 8: Recover Fleet GitOps State

After Rancher recovery, Fleet should resume GitOps operations:

```bash
# Check Fleet agent status on downstream cluster
kubectl get pods -n cattle-fleet-system

# If Fleet agent is stuck, restart it
kubectl rollout restart deployment fleet-agent \
  -n cattle-fleet-system

# Check GitRepo sync status
kubectl get gitrepos.fleet.cattle.io -A

# Force a reconcile
kubectl annotate gitrepo my-repo \
  fleet.cattle.io/force-reconcile="$(date)" \
  -n fleet-default
```

## Conclusion

Downstream clusters are resilient to Rancher management server failures-workloads keep running. Recovery focuses on re-establishing the management connection between Rancher and each downstream cluster. Most clusters reconnect automatically after Rancher is restored. For those that don't, re-importing via the cluster import YAML is a reliable fallback.
