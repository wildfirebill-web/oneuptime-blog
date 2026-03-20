# How to Configure Longhorn Node and Disk Settings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, Configuration, Nodes, Disks

Description: Learn how to configure Longhorn node and disk settings including storage allocation, scheduling policies, and disk management for optimal cluster storage performance.

## Introduction

Longhorn's performance and behavior are significantly influenced by how nodes and disks are configured. Each Longhorn node manages one or more disks, and you can control which disks are used for storage, how much space is allocated, and which workloads can be scheduled on each node. This guide covers all aspects of Longhorn node and disk configuration.

## Understanding Longhorn Node Architecture

Each Kubernetes worker node that runs Longhorn has:
- A Longhorn manager pod managing the node
- One or more disks where replica data is stored
- Configurable scheduling tags for replica placement

## Viewing Node Status

```bash
# List all Longhorn nodes and their status
kubectl get nodes.longhorn.io -n longhorn-system

# Detailed node information
kubectl describe nodes.longhorn.io -n longhorn-system

# Check node disk usage
kubectl get nodes.longhorn.io -n longhorn-system \
  -o custom-columns="NAME:.metadata.name,SCHEDULABLE:.spec.allowScheduling,STORAGE:.status.diskStatus"
```

## Enabling/Disabling Scheduling on a Node

You may want to disable storage scheduling on a node (e.g., for maintenance):

### Via Longhorn UI

1. Navigate to **Node**
2. Find the node you want to configure
3. Click the three-dot menu (⋮)
4. Select **Edit Node and Disks**
5. Toggle **Node Scheduling** to enable or disable

### Via kubectl

```bash
# Disable scheduling on a node (maintenance mode)
kubectl patch nodes.longhorn.io worker-node-1 \
  -n longhorn-system \
  --type merge \
  -p '{"spec": {"allowScheduling": false}}'

# Re-enable scheduling after maintenance
kubectl patch nodes.longhorn.io worker-node-1 \
  -n longhorn-system \
  --type merge \
  -p '{"spec": {"allowScheduling": true}}'
```

## Configuring Disk Settings

### View Current Disk Configuration

```bash
# Get disk details for a node
kubectl get nodes.longhorn.io worker-node-1 \
  -n longhorn-system -o yaml | grep -A 20 "spec:"
```

### Add a Disk to a Longhorn Node

```bash
# SSH to the node and identify available disks
lsblk
df -h

# Create and mount the disk (example with /dev/sdb)
# Format the disk
mkfs.ext4 /dev/sdb

# Create the mount point
mkdir -p /mnt/longhorn-disk1

# Add to /etc/fstab for persistent mount
echo '/dev/sdb /mnt/longhorn-disk1 ext4 defaults 0 0' >> /etc/fstab

# Mount the disk
mount -a
```

Then configure the disk in Longhorn:

```yaml
# node-with-disk.yaml - Configure a node with multiple disks
apiVersion: longhorn.io/v1beta2
kind: Node
metadata:
  name: worker-node-1
  namespace: longhorn-system
spec:
  allowScheduling: true
  disks:
    # Default disk (usually /var/lib/longhorn)
    default-disk:
      allowScheduling: true
      evictionRequested: false
      path: /var/lib/longhorn
      # Reserve 25% for the OS and other uses
      storageReserved: 5368709120   # 5 GiB in bytes
      tags: []
    # Additional SSD disk
    ssd-disk:
      allowScheduling: true
      evictionRequested: false
      path: /mnt/longhorn-disk1
      storageReserved: 1073741824  # 1 GiB reserved
      tags:
        - ssd
        - fast
```

```bash
kubectl apply -f node-with-disk.yaml
```

### Remove a Disk from Longhorn

Before removing a disk, you must evict all replicas from it:

```bash
# Step 1: Request disk eviction (moves replicas off the disk)
kubectl patch nodes.longhorn.io worker-node-1 \
  -n longhorn-system \
  --type merge \
  -p '{"spec": {"disks": {"ssd-disk": {"evictionRequested": true}}}}'

# Step 2: Wait for replicas to be evicted
kubectl get replicas.longhorn.io -n longhorn-system \
  -l longhornnode=worker-node-1 -w

# Step 3: Once no replicas remain on the disk, disable scheduling
kubectl patch nodes.longhorn.io worker-node-1 \
  -n longhorn-system \
  --type merge \
  -p '{"spec": {"disks": {"ssd-disk": {"allowScheduling": false}}}}'
```

## Adding Node and Disk Tags

Tags enable precise replica placement control:

```bash
# Add tags to a node (marks the node as suitable for production workloads)
kubectl patch nodes.longhorn.io worker-node-1 \
  -n longhorn-system \
  --type merge \
  -p '{"spec": {"tags": ["production", "high-memory"]}}'

# Tags are set per-disk in the disk spec (shown in the YAML above)
# Use tags in StorageClass to target specific nodes/disks
```

## Configuring Storage Reservation

Reserve disk space to prevent Longhorn from filling up the disk entirely:

```bash
# Set disk-level storage reservation (bytes)
# This reserves 10 GiB on the default disk of worker-node-1
kubectl patch nodes.longhorn.io worker-node-1 \
  -n longhorn-system \
  --type merge \
  -p '{"spec": {"disks": {"default-disk": {"storageReserved": 10737418240}}}}'
```

## Evicting a Node

When you need to remove a node from Longhorn (e.g., for decommissioning):

```bash
# Step 1: Cordon the Kubernetes node to prevent new pods
kubectl cordon worker-node-1

# Step 2: Evict all replicas from the Longhorn node
kubectl patch nodes.longhorn.io worker-node-1 \
  -n longhorn-system \
  --type merge \
  -p '{"spec": {"evictionRequested": true}}'

# Step 3: Wait for all replicas to be migrated
kubectl get replicas.longhorn.io -n longhorn-system \
  -l longhornnode=worker-node-1

# Step 4: Once no replicas remain, drain the Kubernetes node
kubectl drain worker-node-1 --ignore-daemonsets --delete-emptydir-data

# Step 5: Delete the node
kubectl delete node worker-node-1
```

## Monitoring Node Health

```bash
# Check overall node storage health
kubectl get nodes.longhorn.io -n longhorn-system -o yaml | \
  grep -E "storageAvailable|storageScheduled|storageMaximum"

# Check for nodes with low available storage
kubectl get nodes.longhorn.io -n longhorn-system \
  -o custom-columns="NAME:.metadata.name,AVAILABLE:.status.diskStatus.*.storageAvailable" 2>/dev/null
```

## Conclusion

Proper node and disk configuration is fundamental to Longhorn's performance and reliability. By carefully managing disk allocation, storage reservations, scheduling policies, and node tags, you can ensure that Longhorn uses your cluster resources efficiently while maintaining the replica distribution policies needed for high availability. Regular monitoring of node and disk health helps identify capacity issues before they impact workloads.
