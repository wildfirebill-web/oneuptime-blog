# How to Remove Worker Nodes from K3s

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kubernetes, Cluster Management, DevOps, Linux

Description: Learn how to safely remove worker nodes from a K3s cluster by draining workloads, deleting the node, and cleaning up the agent.

## Introduction

Removing a worker node from a K3s cluster is a two-phase process: first, gracefully migrating workloads off the node using Kubernetes' drain mechanism, then cleaning up the K3s agent from the node itself. Properly following both phases ensures no workload disruption and no stale node entries in your cluster.

## Prerequisites

- `kubectl` configured to communicate with your K3s server
- SSH access to the worker node being removed
- Root/sudo privileges on the worker node

## Step 1: Identify the Node to Remove

```bash
# List all nodes with their status and roles

kubectl get nodes -o wide

# Example output:
# NAME         STATUS   ROLES                  AGE   VERSION
# k3s-server   Ready    control-plane,master   15d   v1.29.3+k3s1
# worker-01    Ready    <none>                 10d   v1.29.3+k3s1
# worker-02    Ready    <none>                 10d   v1.29.3+k3s1

# Check what workloads are running on the target node
kubectl get pods -A --field-selector spec.nodeName=worker-01
```

## Step 2: Cordon the Node

Cordoning prevents new pods from being scheduled on the node:

```bash
# Mark the node as unschedulable
kubectl cordon worker-01

# Verify the node is cordoned
kubectl get node worker-01
# NAME        STATUS                     ROLES    AGE   VERSION
# worker-01   Ready,SchedulingDisabled   <none>   10d   v1.29.3+k3s1
```

## Step 3: Drain the Node

Draining evicts all running pods from the node, triggering Kubernetes to reschedule them on other nodes:

```bash
# Drain all pods from the node
kubectl drain worker-01 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --timeout=120s

# Flags explained:
# --ignore-daemonsets: Skip DaemonSet-managed pods (they run on every node)
# --delete-emptydir-data: Delete pods using emptyDir volumes
# --force: Remove pods not managed by a controller
# --timeout: Time to wait for pod eviction
```

### Handling Drain Failures

If drain fails due to PodDisruptionBudgets:

```bash
# Check for PodDisruptionBudgets blocking the drain
kubectl get pdb -A

# Temporarily disable a PDB if safe to do so
kubectl patch pdb <pdb-name> -n <namespace> \
  -p '{"spec":{"minAvailable":0}}'

# Retry the drain
kubectl drain worker-01 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force

# Restore the PDB afterwards
kubectl patch pdb <pdb-name> -n <namespace> \
  -p '{"spec":{"minAvailable":1}}'
```

## Step 4: Verify Pods Have Been Rescheduled

Ensure critical workloads are running on other nodes:

```bash
# Check that the target node has no non-DaemonSet pods
kubectl get pods -A --field-selector spec.nodeName=worker-01 \
  | grep -v DaemonSet

# Verify pods are running on other nodes
kubectl get pods -A -o wide | grep -v worker-01
```

## Step 5: Delete the Node from Kubernetes

Remove the node object from the cluster:

```bash
# Delete the node from Kubernetes
kubectl delete node worker-01

# Verify the node is removed
kubectl get nodes
```

## Step 6: Uninstall K3s Agent from the Node

SSH to the removed node and run the uninstall script:

```bash
ssh user@worker-01-ip

# Run the K3s agent uninstall script
/usr/local/bin/k3s-agent-uninstall.sh
```

If the uninstall script is not available, manually remove the agent:

```bash
# Stop and disable the service
systemctl stop k3s-agent
systemctl disable k3s-agent

# Remove service files
rm -f /etc/systemd/system/k3s-agent.service
rm -f /etc/systemd/system/k3s-agent.service.env
systemctl daemon-reload

# Remove binary and data
rm -f /usr/local/bin/k3s
rm -rf /var/lib/rancher/k3s
rm -rf /etc/rancher/k3s
rm -rf /run/k3s
rm -rf /var/lib/kubelet
rm -rf /etc/cni/net.d
rm -rf /var/lib/cni
```

## Step 7: Clean Up Network Artifacts

Remove network interfaces created by K3s on the removed node:

```bash
# Remove flannel and bridge interfaces
ip link delete flannel.1 2>/dev/null || true
ip link delete cni0 2>/dev/null || true

# Flush iptables rules
iptables -F && iptables -X
iptables -t nat -F && iptables -t nat -X
iptables -t mangle -F && iptables -t mangle -X
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
```

## Step 8: Remove Node Password Entry on Server

K3s stores per-node passwords on the server. Clean these up:

```bash
# On the K3s server
# Remove the node password file for the removed node
rm -f /var/lib/rancher/k3s/server/cred/node-passwd

# Or selectively remove the entry for the specific node
# The password file maps node names to hashed passwords
```

## Step 9: Verify Cluster Health

After removing the node, verify the remaining cluster is healthy:

```bash
# Check remaining nodes are Ready
kubectl get nodes

# Ensure all critical workloads are running
kubectl get deployments -A
kubectl get pods -A | grep -v Running | grep -v Completed

# Check cluster has enough capacity for workloads
kubectl top nodes
kubectl describe nodes | grep -A 4 "Allocated resources"
```

## Automating Node Removal with a Script

```bash
#!/bin/bash
# remove-k3s-node.sh <node-name> <node-ip>
set -euo pipefail

NODE_NAME="$1"
NODE_IP="$2"

echo "Removing node: $NODE_NAME ($NODE_IP)"

# Cordon the node
kubectl cordon "$NODE_NAME"

# Drain the node
kubectl drain "$NODE_NAME" \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --timeout=120s

# Delete from Kubernetes
kubectl delete node "$NODE_NAME"

# Uninstall K3s from the node
ssh "root@$NODE_IP" "/usr/local/bin/k3s-agent-uninstall.sh"

echo "Node $NODE_NAME successfully removed"
```

## Conclusion

Removing a K3s worker node safely requires draining workloads before deletion to avoid disruption. The drain-delete-uninstall sequence ensures both the Kubernetes control plane and the physical node are cleaned up properly. For production clusters, always verify workload redistribution after removal to ensure your remaining nodes have sufficient capacity to handle the increased load.
