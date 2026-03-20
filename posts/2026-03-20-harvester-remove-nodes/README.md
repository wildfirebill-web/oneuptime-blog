# How to Remove Nodes from Harvester Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Cluster, Maintenance

Description: Learn how to safely remove nodes from a Harvester cluster for decommissioning, hardware replacement, or cluster downsizing.

## Introduction

Removing a node from a Harvester cluster requires careful preparation to ensure VMs are migrated, storage data is replicated to remaining nodes, and the Kubernetes and etcd configurations are updated correctly. Improper node removal can result in data loss or cluster instability. This guide covers the complete safe node removal process.

## Important Considerations

- **Minimum cluster size**: A 3-node cluster is the minimum for HA. Removing a node from a 3-node cluster leaves you with a single point of failure.
- **etcd quorum**: Harvester uses etcd for cluster state. You must maintain an odd number of nodes (3, 5, 7) for quorum. Never remove a node that would break quorum.
- **Storage replicas**: Longhorn volumes default to 3 replicas. Removing a node with replicas will trigger re-replication to remaining nodes.

## Pre-Removal Checklist

```bash
# 1. Check cluster node count

kubectl get nodes | wc -l
# Must leave at least 3 nodes after removal

# 2. Check VMs running on the target node
TARGET_NODE="harvester-node-04"
kubectl get vmi -A --field-selector spec.nodeName=${TARGET_NODE}

# 3. Check Longhorn replicas on the target node
kubectl get replicas.longhorn.io -n longhorn-system \
    -o json | jq --arg node ${TARGET_NODE} \
    '.items[] | select(.spec.nodeID == $node) | .metadata.name'

# 4. Check total cluster storage capacity
kubectl get node ${TARGET_NODE} \
    -o jsonpath='{.status.capacity}'

# Ensure remaining nodes have enough storage for all replica data
```

## Step 1: Enable Maintenance Mode

Maintenance mode migrates VMs off the node and stops new scheduling:

### Via the UI

1. Navigate to **Hosts** in Harvester
2. Find the node to remove
3. Click the **⋮** menu → **Enable Maintenance Mode**
4. Harvester will live-migrate all VMs off the node
5. Wait for the node status to show **Maintenance**

### Via kubectl

```bash
TARGET_NODE="harvester-node-04"

# Enable maintenance mode via annotation
kubectl annotate node ${TARGET_NODE} \
    harvesterhci.io/maintenance-mode="true"

# This triggers VM live migration off the node
# Monitor VM migrations
kubectl get vmi -A --field-selector spec.nodeName=${TARGET_NODE} -w

# Wait until no VMs remain on the node
while kubectl get vmi -A \
    --field-selector spec.nodeName=${TARGET_NODE} \
    --no-headers | grep -q "."; do
    echo "Waiting for VMs to migrate..."
    sleep 30
done
echo "All VMs migrated off ${TARGET_NODE}"
```

## Step 2: Drain the Node

After VMs are migrated, drain remaining Kubernetes workloads:

```bash
# Cordon the node (prevent new scheduling)
kubectl cordon ${TARGET_NODE}

# Drain non-VM workloads
kubectl drain ${TARGET_NODE} \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --timeout=300s

# Verify no user workloads remain
kubectl get pods -A \
    --field-selector spec.nodeName=${TARGET_NODE} | grep -v daemonsets
```

## Step 3: Evacuate Longhorn Replicas

Before removing the node, evacuate its Longhorn replicas to maintain data redundancy:

```bash
# Disable disk scheduling on the target node
kubectl patch node.longhorn.io ${TARGET_NODE} \
    -n longhorn-system \
    --type merge \
    -p '{"spec":{"allowScheduling":false}}'

# Evict all replicas from the node
# This triggers replica rebuilding on other nodes
kubectl patch node.longhorn.io ${TARGET_NODE} \
    -n longhorn-system \
    --type merge \
    -p '{"spec":{"evictionRequested":true}}'

# Monitor replica evacuation
# Wait for all replicas to be moved off the node
watch kubectl get replicas.longhorn.io -n longhorn-system \
    -o json | jq --arg node ${TARGET_NODE} \
    '.items | map(select(.spec.nodeID == $node)) | length'

# When the count reaches 0, all replicas have been evacuated
echo "Waiting for replica evacuation to complete..."
while [ "$(kubectl get replicas.longhorn.io -n longhorn-system \
    -o json | jq --arg node ${TARGET_NODE} \
    '[.items[] | select(.spec.nodeID == $node)] | length')" -gt 0 ]; do
    sleep 30
    echo "Still evacuating replicas..."
done
echo "All replicas evacuated!"
```

## Step 4: Remove the Node from the Cluster

### Via the Harvester UI

1. Navigate to **Hosts**
2. Find the node in maintenance mode
3. Click the **⋮** menu → **Delete**
4. Confirm the deletion

### Via kubectl

```bash
# Remove the node from Kubernetes
kubectl delete node ${TARGET_NODE}

# Remove from Longhorn
kubectl delete node.longhorn.io ${TARGET_NODE} -n longhorn-system

# Remove the RKE2 cluster member (on a surviving node)
# Get the etcd member ID for the node being removed
kubectl exec -n kube-system \
    $(kubectl get pods -n kube-system -l component=etcd \
        --field-selector spec.nodeName=harvester-node-01 -o name) -- \
    etcdctl --endpoints=https://127.0.0.1:2379 \
            --cacert /var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
            --cert /var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
            --key /var/lib/rancher/rke2/server/tls/etcd/server-client.key \
            member list

# Remove the etcd member
# Replace MEMBER_ID with the ID shown in the previous command
kubectl exec -n kube-system \
    $(kubectl get pods -n kube-system -l component=etcd \
        --field-selector spec.nodeName=harvester-node-01 -o name) -- \
    etcdctl --endpoints=https://127.0.0.1:2379 \
            --cacert /var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
            --cert /var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
            --key /var/lib/rancher/rke2/server/tls/etcd/server-client.key \
            member remove <MEMBER_ID>
```

## Step 5: Clean Up the Removed Node

If you plan to reuse the node hardware:

```bash
# SSH into the removed node (if still accessible)
ssh rancher@192.168.1.14

# Stop RKE2 services
sudo systemctl stop rke2-server

# Clean up RKE2 data
sudo /usr/local/bin/rke2-uninstall.sh

# Or manually clean data directories
sudo rm -rf /var/lib/rancher/rke2
sudo rm -rf /etc/rancher/rke2
sudo rm -rf /run/k3s
```

## Step 6: Post-Removal Verification

```bash
# Verify the node is gone
kubectl get nodes

# Check etcd cluster is healthy with remaining nodes
kubectl get componentstatuses

# Verify Longhorn is rebalancing
kubectl get volumes.longhorn.io -n longhorn-system \
    -o jsonpath='{range .items[*]}{.metadata.name}: {.status.robustness}{"\n"}{end}'
# All should show: healthy

# Check replica counts are being restored
kubectl get replicas.longhorn.io -n longhorn-system \
    -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeID,STATE:.status.currentState'

# Verify all VMs are running on remaining nodes
kubectl get vmi -A -o custom-columns='NAME:.metadata.name,NODE:.status.nodeName'
```

## Conclusion

Safely removing a node from a Harvester cluster requires a systematic approach: migrate VMs, evacuate Longhorn replicas, drain Kubernetes workloads, then remove. Rushing any of these steps can result in VM downtime or data loss. The key safeguard is ensuring Longhorn replica evacuation completes before node deletion - this ensures your data has 3 replicas on the remaining nodes before any single node is removed. After removal, Longhorn automatically rebuilds replicas to maintain the configured replica count on the remaining nodes.
