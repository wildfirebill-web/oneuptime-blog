# How to Perform Rolling Cluster Upgrades in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Upgrade, Operations, RKE2

Description: Perform safe rolling Kubernetes version upgrades on Rancher-managed clusters with zero downtime using RKE2 and Rancher's built-in upgrade management.

## Introduction

Keeping Kubernetes clusters up to date is critical for security patches, feature access, and support. Rancher provides guided cluster upgrade workflows for RKE2 and K3s clusters, with the ability to control the upgrade pace (one node at a time) and monitor progress. This guide covers how to plan and execute rolling cluster upgrades safely.

## Pre-Upgrade Checklist

Before upgrading, verify:

```bash
# 1. Check current cluster and node versions

kubectl get nodes -o wide
kubectl version

# 2. Check available upgrade versions in Rancher
# Rancher UI: Cluster → Edit Config → Kubernetes Version dropdown

# 3. Verify etcd health (for RKE2 clusters)
sudo /var/lib/rancher/rke2/bin/etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  endpoint health

# 4. Verify all nodes are Ready
kubectl get nodes

# 5. Check for any PodDisruptionBudgets that could block drain
kubectl get pdb -A

# 6. Verify workload replicas (ensure > 1 for critical services)
kubectl get deployments -A -o json \
  | jq '.items[] | select(.spec.replicas < 2) | "\(.metadata.namespace)/\(.metadata.name)"'
```

## Step 1: Take an etcd Snapshot Before Upgrading

```bash
# Create a manual snapshot before the upgrade
# In Rancher UI: Cluster → Snapshots → Create Snapshot

# Or via kubectl (for RKE2):
kubectl create job --from=cronjob/rke2-etcd-snapshot-now \
  manual-snapshot-pre-upgrade \
  -n kube-system

# Verify snapshot was created
kubectl get jobs -n kube-system manual-snapshot-pre-upgrade
```

## Step 2: Initiate the Upgrade via Rancher UI

1. Navigate to **Cluster Management** → select the cluster.
2. Click **⋮ → Edit Config**.
3. Under **Kubernetes Version**, select the target version.
4. Under **Upgrade Strategy**, configure:
   - **Max Unavailable Workers**: `1` (safest)
   - **Control Plane Concurrency**: `1`
   - **Enable Drain Before Upgrade**: Yes
5. Click **Save**.

Rancher will begin the upgrade process automatically.

## Step 3: Initiate the Upgrade via API

```bash
# Get the cluster ID
CLUSTER_ID=$(curl -sk \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "https://rancher.example.com/v3/clusters?name=my-cluster" \
  | jq -r '.data[0].id')

# Initiate the upgrade
curl -sk -X PUT \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"kubernetesVersion\": \"v1.29.0+rke2r1\",
    \"rkeConfig\": {
      \"upgradeStrategy\": {
        \"controlPlaneDrainOptions\": {
          \"enabled\": true,
          \"timeout\": 120
        },
        \"workerDrainOptions\": {
          \"enabled\": true,
          \"timeout\": 120
        },
        \"workerConcurrency\": \"1\"
      }
    }
  }" \
  "https://rancher.example.com/v3/clusters/${CLUSTER_ID}"
```

## Step 4: Monitor the Upgrade Progress

```bash
# Watch node versions change during the rolling upgrade
watch kubectl get nodes -o custom-columns=\
"NAME:.metadata.name,ROLE:.metadata.labels['node-role.kubernetes.io/control-plane'],\
STATUS:.status.conditions[-1].type,VERSION:.status.nodeInfo.kubeletVersion"

# Check the upgrade machine plan
kubectl get rkecontrolplanes -n fleet-default
kubectl describe rkecontrolplanes -n fleet-default <cluster-name>

# Check specific node machine status
kubectl get machines -n fleet-default
kubectl describe machine -n fleet-default <machine-name>

# Watch Rancher server logs for upgrade events
kubectl logs -n cattle-system -l app=rancher -f --tail=100 \
  | grep -iE "upgrade|version|machine"
```

## Step 5: Understand the Upgrade Order

For an RKE2 cluster, the upgrade order is:

1. **etcd nodes** are upgraded first (one at a time).
2. **Control plane nodes** are upgraded next (one at a time).
3. **Worker nodes** are upgraded last (configurable concurrency).

Each node is:
1. Cordoned (no new pods scheduled).
2. Drained (existing pods evicted gracefully).
3. Upgraded (RKE2 binary replaced, service restarted).
4. Uncordoned (returns to the scheduling pool).

## Step 6: Handle Stuck Upgrades

```bash
# If a node gets stuck in Upgrading state:
kubectl get machine -n fleet-default | grep -v Ready

# Check the machine plan for errors
kubectl describe machine -n fleet-default <stuck-machine>

# Common issues:
# - PodDisruptionBudget blocking drain → temporarily patch the PDB
kubectl patch pdb <pdb-name> -n <namespace> \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/minAvailable","value":0}]'

# - Node drain timeout → increase the timeout
# Edit the cluster via Rancher UI: Cluster → Edit → Drain Timeout: 300s

# - Stale token → force re-registration of the node
```

## Step 7: Post-Upgrade Verification

```bash
# Verify all nodes are on the new version
kubectl get nodes -o custom-columns="NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion"

# Verify all system pods are running
kubectl get pods -n kube-system -o wide
kubectl get pods -n cattle-system -o wide

# Run a quick sanity check deployment
kubectl create deployment upgrade-test --image=nginx:stable --replicas=3
kubectl rollout status deployment/upgrade-test
kubectl delete deployment upgrade-test

# Verify etcd health after upgrade
sudo /var/lib/rancher/rke2/bin/etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  endpoint health
```

## Conclusion

Rolling cluster upgrades in Rancher are designed to minimize downtime and risk. By taking a pre-upgrade snapshot, configuring a conservative upgrade strategy (one node at a time with draining), and monitoring progress through both Rancher UI and kubectl, you can safely keep your clusters current. Always upgrade in the pattern: dev → staging → production, and verify each environment before proceeding to the next.
