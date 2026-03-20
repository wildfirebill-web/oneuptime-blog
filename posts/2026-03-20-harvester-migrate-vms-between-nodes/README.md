# How to Migrate VMs Between Nodes in Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Migration, KubeVirt

Description: Learn how to migrate virtual machines between nodes in Harvester for maintenance, load balancing, and fault tolerance.

## Introduction

VM migration in Harvester allows you to move running or stopped virtual machines from one cluster node to another. This is essential for node maintenance (draining a node before hardware work), load rebalancing, and improving VM placement for performance. Harvester supports both live migration (VM keeps running during the move) and cold migration (VM is stopped during the move).

## Migration Types

| Type | VM State | Downtime | Use Case |
|---|---|---|---|
| Live Migration | Running | None | Node maintenance, load balancing |
| Cold Migration | Stopped | Yes (planned) | Forced relocation |
| Eviction | Running | Brief | Node drain for maintenance |

## Prerequisites

- A multi-node Harvester cluster (minimum 2 nodes)
- Sufficient resources on the target node (CPU, RAM)
- Shared storage (Longhorn) configured across all nodes
- For live migration: Network bandwidth between nodes for memory transfer

## Step 1: Check Current VM Placement

```bash
# See which node each VM is running on

kubectl get vmi -n default \
    -o custom-columns=\
'NAME:.metadata.name,NODE:.status.nodeName,PHASE:.status.phase'

# Example output:
# NAME            NODE               PHASE
# ubuntu-web-01   harvester-node-01  Running
# database-01     harvester-node-02  Running
# app-server-01   harvester-node-01  Running
```

## Step 2: Live Migrate a VM

### Via the UI

1. Navigate to **Virtual Machines**
2. Find the VM you want to migrate
3. Click the **⋮** menu → **Migrate**
4. Select the target node (or let Harvester choose automatically)
5. Click **Confirm**

### Via kubectl

```yaml
# vm-migration.yaml
# Trigger a live migration for a VM

apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceMigration
metadata:
  name: migrate-ubuntu-web-01
  namespace: default
spec:
  # Name of the VMI (VirtualMachineInstance) to migrate
  vmiName: ubuntu-web-01
```

```bash
kubectl apply -f vm-migration.yaml

# Watch the migration progress
kubectl get virtualmachineinstancemigration migrate-ubuntu-web-01 -n default -w

# Migration phases:
# Pending -> Scheduling -> Scheduled -> PreparingTarget -> TargetReady
# -> Running -> Succeeded (or Failed)

# Check which node the VM is now on
kubectl get vmi ubuntu-web-01 -n default \
    -o jsonpath='{.status.nodeName}'
```

### Migration Status Details

```bash
# Get detailed migration status
kubectl describe virtualmachineinstancemigration migrate-ubuntu-web-01 -n default

# Check the VMI migration state
kubectl get vmi ubuntu-web-01 -n default \
    -o jsonpath='{.status.migrationState}' | jq .

# View migration events
kubectl get events -n default \
    --field-selector reason=Migration \
    --sort-by='.lastTimestamp'
```

## Step 3: Migrate to a Specific Node

To target a specific node, use node affinity or node selectors:

```bash
# First, label the target node
kubectl label node harvester-node-03 \
    migration-target=ubuntu-web-01

# Add a node affinity rule to the VM to prefer node-03
kubectl patch vm ubuntu-web-01 -n default --type json \
-p '[{
    "op": "add",
    "path": "/spec/template/spec/affinity",
    "value": {
        "nodeAffinity": {
            "preferredDuringSchedulingIgnoredDuringExecution": [{
                "weight": 100,
                "preference": {
                    "matchExpressions": [{
                        "key": "kubernetes.io/hostname",
                        "operator": "In",
                        "values": ["harvester-node-03"]
                    }]
                }
            }]
        }
    }
}]'

# Trigger the migration
kubectl apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceMigration
metadata:
  name: migrate-to-node03
  namespace: default
spec:
  vmiName: ubuntu-web-01
EOF
```

## Step 4: Drain a Node for Maintenance

To migrate all VMs off a node before maintenance:

```bash
# Cordon the node to prevent new scheduling
kubectl cordon harvester-node-01

# Evict all VMs from the node
# This triggers live migrations for all migratable VMs
kubectl drain harvester-node-01 \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --pod-selector 'vm.kubevirt.io/name' \
    --disable-eviction=false

# Watch VMs migrating off the node
watch kubectl get vmi -n default \
    -o custom-columns='NAME:.metadata.name,NODE:.status.nodeName'

# Alternatively, use virtctl to evict VMs
virtctl migrate --all -n default \
    --node harvester-node-01
```

```bash
# After maintenance, uncordon the node
kubectl uncordon harvester-node-01

# Verify the node is ready to accept workloads
kubectl get node harvester-node-01
```

## Step 5: Bulk Migrate VMs with a Script

```bash
#!/bin/bash
# bulk-migrate.sh - Migrate all VMs from one node to another

SOURCE_NODE="harvester-node-01"
NAMESPACE="default"

echo "Finding VMs on ${SOURCE_NODE}..."

# Get all VMIs on the source node
VMs=$(kubectl get vmi -n ${NAMESPACE} \
    -o jsonpath="{.items[?(@.status.nodeName==\"${SOURCE_NODE}\")].metadata.name}")

if [ -z "$VMs" ]; then
    echo "No VMs found on ${SOURCE_NODE}"
    exit 0
fi

echo "VMs to migrate: ${VMs}"

# Trigger migration for each VM
for VM in $VMs; do
    echo "Migrating ${VM}..."
    kubectl apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceMigration
metadata:
  name: mig-${VM}-$(date +%s)
  namespace: ${NAMESPACE}
spec:
  vmiName: ${VM}
EOF
    # Small delay to avoid overwhelming the cluster
    sleep 5
done

echo "All migrations initiated. Monitoring..."

# Wait for all VMs to leave the source node
while kubectl get vmi -n ${NAMESPACE} \
    -o jsonpath="{.items[?(@.status.nodeName==\"${SOURCE_NODE}\")].metadata.name}" \
    | grep -q "."; do
    echo "VMs still on ${SOURCE_NODE}, waiting..."
    sleep 10
done

echo "All VMs migrated from ${SOURCE_NODE}"
```

## Troubleshooting Migration Issues

```bash
# Migration stuck in Pending:
# Check resource availability on target nodes
kubectl describe node harvester-node-02 | grep -A 10 "Conditions:"
kubectl describe node harvester-node-02 | grep -A 20 "Allocated resources"

# Migration failed with network error:
# Check that migration network is reachable between nodes
kubectl get kubevirt -n harvester-system -o yaml | \
    grep -A 10 "migrations:"

# Check the migration pod logs
kubectl get pods -n default | grep virt-launcher
kubectl logs -n default \
    $(kubectl get pods -n default -l "vm.kubevirt.io/name=ubuntu-web-01" -o name)
```

## Conclusion

VM migration in Harvester is a powerful capability that enables zero-downtime maintenance and flexible workload placement. Live migration allows VMs to continue serving traffic while moving between nodes - an essential feature for production environments with strict availability requirements. By combining node cordoning, draining, and targeted migration policies, you can maintain your cluster infrastructure without scheduling maintenance windows that impact users. Always verify sufficient resources on the destination nodes before initiating large-scale migrations.
