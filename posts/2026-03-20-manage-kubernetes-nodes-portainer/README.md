# How to Manage Kubernetes Nodes from Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Node Management, Cluster Administration, DevOps

Description: Learn how to view, inspect, cordon, drain, and manage Kubernetes nodes directly from the Portainer UI.

## Accessing Nodes in Portainer

1. Select your Kubernetes environment in Portainer.
2. In the left sidebar, go to **Cluster > Nodes**.
3. The node list shows all nodes with their status, roles, and resource usage.

## Node Status Indicators

| Status | Meaning |
|--------|---------|
| Ready | Node is healthy and can schedule pods |
| NotReady | Node cannot schedule pods (check conditions) |
| SchedulingDisabled | Node is cordoned |

## Inspecting a Node

Click a node name to view:
- OS and container runtime details
- CPU/memory capacity and allocatable resources
- Node conditions (MemoryPressure, DiskPressure, PIDPressure)
- Labels and annotations
- Running pods and their resource consumption

```bash
# CLI equivalent
kubectl describe node <node-name>

# Get node conditions only
kubectl get node <node-name> \
  -o jsonpath='{.status.conditions[*].type}={.status.conditions[*].status}'
```

## Cordoning a Node (Disable Scheduling)

Cordoning prevents new pods from being scheduled on a node without evicting existing pods:

```bash
# Cordon a node via CLI
kubectl cordon <node-name>

# Uncordon a node to re-enable scheduling
kubectl uncordon <node-name>
```

In Portainer, click the node and look for the **Cordon** action button.

## Draining a Node (Evict All Pods)

Draining first cordons the node, then evicts all pods (except DaemonSets and local storage pods):

```bash
# Drain a node for maintenance
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=30

# After maintenance, uncordon the node
kubectl uncordon <node-name>
```

## Adding Labels to Nodes

Labels are used for pod scheduling constraints:

```bash
# Add a label to a node
kubectl label node <node-name> environment=production

# Remove a label
kubectl label node <node-name> environment-

# View labels on all nodes
kubectl get nodes --show-labels
```

## Adding Taints to Nodes

Taints prevent pods from scheduling on a node unless they have a matching toleration:

```bash
# Add a taint to reserve a node for specific workloads
kubectl taint nodes <node-name> dedicated=gpu:NoSchedule

# Remove the taint
kubectl taint nodes <node-name> dedicated=gpu:NoSchedule-
```

## Checking Node Resource Pressure

```bash
# Check resource usage per node
kubectl top nodes

# Check if any nodes are under pressure
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
MEMORYPRESSURE:.status.conditions[?(@.type=="MemoryPressure")].status,\
DISKPRESSURE:.status.conditions[?(@.type=="DiskPressure")].status
```

## Conclusion

Portainer provides a visual interface for common Kubernetes node management tasks. For operations like draining and cordoning, Portainer abstracts the kubectl commands into clear UI actions — making cluster maintenance accessible to team members who aren't kubectl experts.
