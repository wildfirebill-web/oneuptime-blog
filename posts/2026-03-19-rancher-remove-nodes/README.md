# How to Remove Nodes from a Rancher-Managed Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Cluster Management, Node Management

Description: Learn how to safely remove nodes from a Rancher-managed Kubernetes cluster without disrupting workloads.

Removing nodes from a cluster is necessary during hardware decommissioning, scaling down, or replacing problematic nodes. This guide covers how to safely remove nodes from Rancher-managed clusters while minimizing workload disruption.

## Prerequisites

- Administrative access to the Rancher UI
- `kubectl` access to the target cluster
- Understanding of which workloads are running on the node being removed

## Step 1: Identify the Node to Remove

First, identify the node and understand what is running on it:

```bash
# List all nodes with roles
kubectl get nodes -o wide

# Check what pods are running on the target node
kubectl get pods -A --field-selector spec.nodeName=<NODE_NAME> -o wide

# Check node resource usage
kubectl top node <NODE_NAME>
```

## Step 2: Verify Cluster Capacity

Before removing a node, ensure the remaining nodes have enough capacity:

```bash
# Check overall cluster capacity
kubectl top nodes

# Check resource requests vs capacity
kubectl describe nodes | grep -A5 "Allocated resources"
```

Ensure that:

- Remaining nodes have enough CPU and memory for redistributed workloads
- Pod disruption budgets will be respected
- No critical workloads will lose all replicas

## Step 3: Cordon the Node

Cordoning prevents new pods from being scheduled on the node:

### Via the Rancher UI

1. Go to your cluster
2. Navigate to **Nodes**
3. Click the three-dot menu next to the target node
4. Select **Cordon**

### Via kubectl

```bash
kubectl cordon <NODE_NAME>
```

Verify the node is cordoned:

```bash
kubectl get node <NODE_NAME>
# STATUS should show SchedulingDisabled
```

## Step 4: Drain the Node

Draining evicts all pods from the node, respecting pod disruption budgets:

### Via the Rancher UI

1. Click the three-dot menu next to the cordoned node
2. Select **Drain**
3. Configure drain options:
   - **Grace Period**: Time to wait for pods to terminate (default: 30 seconds)
   - **Timeout**: Maximum time for the drain operation
   - **Ignore DaemonSets**: Should be enabled (DaemonSet pods cannot be rescheduled)
   - **Delete Empty Dir Data**: Enable if pods use emptyDir volumes

### Via kubectl

```bash
kubectl drain <NODE_NAME> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=300s
```

### Handling Drain Failures

If the drain gets stuck, check which pods are blocking it:

```bash
kubectl get pods -A --field-selector spec.nodeName=<NODE_NAME>
```

Common blockers:

- **Pods with PDB violations**: Ensure enough replicas exist on other nodes
- **Pods with local storage**: Use `--delete-emptydir-data` or `--force`
- **Standalone pods**: Pods not managed by a controller. Use `--force` to delete them

```bash
# Force drain (use with caution)
kubectl drain <NODE_NAME> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=30
```

## Step 5: Remove the Node

### From the Rancher UI

1. After draining, click the three-dot menu on the node
2. Select **Delete**
3. Confirm the deletion

### From kubectl

```bash
kubectl delete node <NODE_NAME>
```

### For Node-Driver Provisioned Clusters

If the node was provisioned by a Rancher node driver:

1. Go to **Cluster Management**
2. Click **Edit Config** on the cluster
3. Reduce the machine count in the appropriate pool
4. Click **Save**

Rancher will drain and remove the node automatically and clean up the infrastructure (VM deletion).

### For Cloud-Managed Clusters

For EKS, GKE, or AKS clusters, reduce the node group or pool size through the cluster configuration in Rancher.

## Step 6: Clean Up the Removed Node

If the node was part of a custom cluster, clean up the Rancher agent and Kubernetes components on the removed machine:

```bash
# Stop and disable Rancher system agent
sudo systemctl stop rancher-system-agent
sudo systemctl disable rancher-system-agent

# For RKE2 nodes
sudo /usr/local/bin/rke2-uninstall.sh

# For K3s nodes
sudo /usr/local/bin/k3s-uninstall.sh
# or for agent nodes
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

Clean up remaining data:

```bash
sudo rm -rf /var/lib/rancher
sudo rm -rf /etc/rancher
sudo rm -rf /var/lib/kubelet
sudo rm -rf /etc/kubernetes
```

## Step 7: Verify Workload Health

After removing the node, verify all workloads are healthy:

```bash
# Check for pods in non-Running state
kubectl get pods -A | grep -v Running | grep -v Completed

# Verify deployments have all replicas
kubectl get deployments -A

# Check events for any issues
kubectl get events -A --sort-by='.lastTimestamp' | tail -20
```

## Removing etcd Nodes

Removing etcd nodes requires extra caution to maintain cluster quorum.

### Rules for etcd Node Removal

- Never reduce below 3 etcd nodes in production
- Always maintain an odd number of etcd nodes
- Remove only one etcd node at a time
- Verify etcd health after each removal

### Check etcd Health Before Removal

```bash
# For RKE2
sudo /var/lib/rancher/rke2/bin/etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  member list
```

### Remove the etcd Member

1. Remove the member from etcd first
2. Then remove the node from Kubernetes
3. Verify etcd cluster health after removal

```bash
# Remove etcd member
sudo /var/lib/rancher/rke2/bin/etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  member remove <MEMBER_ID>

# Verify health
sudo /var/lib/rancher/rke2/bin/etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  endpoint health
```

## Removing Control Plane Nodes

When removing control plane nodes:

- Maintain at least 2 control plane nodes (3 recommended)
- Ensure the load balancer is updated to remove the node's IP
- Verify API server accessibility after removal

## Troubleshooting

- **Drain timeout**: Increase the timeout or identify stuck pods. Use `--force` as a last resort.
- **Node still appears after deletion**: Check for finalizers on the node object. Remove them if the node is truly gone.
- **Workloads not rescheduling**: Check resource availability on remaining nodes and verify there are no node affinity rules preventing scheduling.
- **etcd quorum issues**: Never remove more than one etcd node at a time. Restore from backup if quorum is lost.

## Conclusion

Removing nodes from a Rancher-managed cluster follows a careful sequence: cordon the node to stop scheduling, drain it to evict workloads safely, delete it from the cluster, and clean up the machine. Always verify that remaining nodes have enough capacity, respect pod disruption budgets during draining, and take special care when removing etcd or control plane nodes to maintain cluster availability.
