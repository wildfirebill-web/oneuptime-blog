# How to Configure Longhorn Volume Replication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, Replication, High Availability

Description: Understand and configure Longhorn's volume replication system to ensure data durability and high availability for your Kubernetes storage.

## Introduction

Longhorn's replication is one of its core features. Each Longhorn volume maintains multiple replicas — copies of the data — distributed across different nodes. If a node fails, Longhorn can automatically rebuild the replica on another healthy node, ensuring your data remains safe and accessible. This guide explains how to configure and manage volume replication.

## How Longhorn Replication Works

Longhorn creates multiple replicas of each volume and distributes them across nodes. Key characteristics:

- Each write operation is synchronously replicated to all replicas
- Longhorn can tolerate `n-1` replica failures (where `n` is the replica count)
- Replicas are scheduled on different nodes to maximize fault tolerance
- When a replica is lost, Longhorn automatically rebuilds it on a healthy node

## Configuring Default Replica Count

### Via Longhorn Settings (Global Default)

```bash
# Patch the Longhorn global setting for default replica count
kubectl patch settings.longhorn.io default-replica-count \
  -n longhorn-system \
  --type merge \
  -p '{"value": "3"}'
```

Or through the Longhorn UI:
1. Navigate to **Setting** → **General**
2. Find **Default Replica Count**
3. Set the value (commonly `2` or `3`)
4. Click **Save**

### Via StorageClass

Set the default replica count per storage class:

```yaml
# storageclass-3replicas.yaml - Storage class with 3 replicas
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-ha
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  # All volumes created from this class will have 3 replicas
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  fsType: "ext4"
```

```bash
kubectl apply -f storageclass-3replicas.yaml
```

## Changing Replica Count for an Existing Volume

### Via Longhorn UI

1. Navigate to **Volume**
2. Click the three-dot menu next to the volume
3. Select **Update Replicas Count**
4. Enter the new replica count
5. Click **OK**

Longhorn will begin creating or removing replicas as needed.

### Via kubectl

```bash
# Update replica count for a specific Longhorn volume
kubectl patch volume.longhorn.io my-volume \
  -n longhorn-system \
  --type merge \
  -p '{"spec": {"numberOfReplicas": 2}}'
```

## Monitoring Replica Status

```bash
# List all replicas for a specific volume
kubectl get replicas.longhorn.io -n longhorn-system \
  -l longhornvolume=my-volume

# Check replica health and location
kubectl describe replicas.longhorn.io -n longhorn-system \
  -l longhornvolume=my-volume
```

Expected output shows each replica with:
- **Spec.NodeID**: The node where the replica lives
- **Status.CurrentState**: `running` or `error`

## Configuring Replica Soft Anti-Affinity

By default, Longhorn tries to schedule replicas on different nodes. The **Replica Soft Anti-Affinity** setting controls this behavior:

```bash
# Enable soft anti-affinity (allows same-node replicas if needed)
kubectl patch settings.longhorn.io replica-soft-anti-affinity \
  -n longhorn-system \
  --type merge \
  -p '{"value": "true"}'

# With soft anti-affinity disabled (strict), Longhorn will refuse
# to create a volume if it cannot place replicas on different nodes
kubectl patch settings.longhorn.io replica-soft-anti-affinity \
  -n longhorn-system \
  --type merge \
  -p '{"value": "false"}'
```

## Configuring Replica Zone Anti-Affinity

For multi-zone clusters, distribute replicas across availability zones:

```bash
# Enable zone anti-affinity for replicas
kubectl patch settings.longhorn.io replica-zone-soft-anti-affinity \
  -n longhorn-system \
  --type merge \
  -p '{"value": "false"}'  # false = strict (recommended for HA)
```

Nodes must be labeled with topology zones:

```bash
# Label nodes with their availability zone
kubectl label node worker-1 topology.kubernetes.io/zone=us-east-1a
kubectl label node worker-2 topology.kubernetes.io/zone=us-east-1b
kubectl label node worker-3 topology.kubernetes.io/zone=us-east-1c
```

## Replica Rebuild Settings

### Concurrent Replica Rebuild Limit

Control how many replicas can be rebuilt simultaneously to limit impact on performance:

```bash
# Set maximum concurrent replica rebuilds per node
kubectl patch settings.longhorn.io concurrent-replica-rebuild-per-node-limit \
  -n longhorn-system \
  --type merge \
  -p '{"value": "5"}'
```

### Replica Replenishment Wait Interval

Set how long Longhorn waits before rebuilding a failed replica (to avoid unnecessary rebuilds from transient failures):

```bash
# Set replica replenishment wait to 600 seconds (10 minutes)
kubectl patch settings.longhorn.io replica-replenishment-wait-interval \
  -n longhorn-system \
  --type merge \
  -p '{"value": "600"}'
```

## Replica Scheduling with Node and Disk Tags

Tag nodes and disks to control replica placement:

```bash
# In Longhorn UI: Node → Edit → Add Tag: "production"
# In Longhorn UI: Disk → Edit → Add Tag: "nvme"
```

Then reference these tags in your StorageClass:

```yaml
# storageclass-tagged.yaml - Schedule replicas on tagged resources
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-nvme
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  # Only place replicas on nodes tagged "production"
  nodeSelector: "production"
  # Only use disks tagged "nvme"
  diskSelector: "nvme"
```

## Checking Replication Health

```bash
# Get overall volume health including replica status
kubectl get volumes.longhorn.io -n longhorn-system

# Check the health of all volumes
kubectl get volumes.longhorn.io -n longhorn-system \
  -o custom-columns="NAME:.metadata.name,STATE:.status.state,HEALTH:.status.robustness,REPLICAS:.spec.numberOfReplicas"
```

Healthy volumes show `healthy` robustness. A degraded volume indicates one or more replicas are unhealthy or rebuilding.

## Conclusion

Longhorn's replication system provides strong data durability guarantees through automatic replica management and rebuilding. By configuring appropriate replica counts, anti-affinity rules, and zone distribution, you can build a storage system that tolerates node and zone failures. Regular monitoring of replica health through the Longhorn UI or `kubectl` helps you catch and address issues before they impact application availability.
