# How to Uninstall RKE2

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Rancher, Administration, Cleanup

Description: Step-by-step instructions to cleanly uninstall RKE2 from server and agent nodes, removing all associated data and configurations.

## Introduction

Whether you are decommissioning a node, rebuilding an environment, or migrating to a different Kubernetes distribution, knowing how to cleanly uninstall RKE2 is important. RKE2 ships with built-in uninstall scripts that handle most of the cleanup automatically. This guide covers the full process for both server and agent nodes.

## Prerequisites

- Root or sudo access on the node
- The node should be drained before uninstalling (for production clusters)

## Step 1: Drain the Node (Production Clusters)

Before uninstalling, gracefully evict all workloads from the node to avoid downtime.

```bash
# From a node with kubectl access, drain the node
# Replace <NODE_NAME> with the actual node name
kubectl drain <NODE_NAME> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force

# Verify all pods have been evicted
kubectl get pods --all-namespaces -o wide | grep <NODE_NAME>
```

## Step 2: Stop the RKE2 Service

```bash
# Stop the server service (on server nodes)
sudo systemctl stop rke2-server.service

# Stop the agent service (on agent nodes)
sudo systemctl stop rke2-agent.service

# Verify the service has stopped
sudo systemctl status rke2-server.service
sudo systemctl status rke2-agent.service
```

## Step 3: Run the Uninstall Script

RKE2 installs uninstall scripts during setup. These scripts stop services, remove binaries, and clean up network interfaces.

### Uninstalling the RKE2 Server

```bash
# Run the server uninstall script
sudo /usr/local/bin/rke2-uninstall.sh
```

### Uninstalling the RKE2 Agent

```bash
# Run the agent uninstall script
sudo /usr/local/bin/rke2-agent-uninstall.sh
```

These scripts perform the following actions automatically:
- Stop and disable the systemd service
- Remove the RKE2 binary
- Delete the containerd shims
- Remove CNI plugins and network interfaces
- Clean up iptables rules

## Step 4: Remove Residual Data

Even after running the uninstall script, some directories may remain if they contain user data or snapshots.

```bash
# Remove the RKE2 data directory (contains etcd data, certificates, etc.)
sudo rm -rf /var/lib/rancher/rke2/

# Remove the RKE2 configuration directory
sudo rm -rf /etc/rancher/rke2/

# Remove the RKE2 log directory
sudo rm -rf /var/log/pods/

# Remove the kubectl config if added to your home directory
rm -f ~/.kube/config
```

## Step 5: Clean Up Network Interfaces

The uninstall script should handle most network cleanup, but verify manually:

```bash
# Check for lingering CNI network interfaces
ip link show | grep -E "cni|flannel|calico|cilium|vxlan"

# Remove any remaining interfaces manually if found
# Example: sudo ip link delete flannel.1
sudo ip link delete <interface_name>

# Check for leftover iptables rules
sudo iptables -L -n | grep -E "KUBE|CNI|FLANNEL"

# Flush all iptables rules if needed (WARNING: this affects all rules)
# sudo iptables -F
# sudo iptables -X
# sudo iptables -t nat -F
# sudo iptables -t nat -X
```

## Step 6: Remove the Node from the Cluster

From a remaining server node, delete the removed node from the Kubernetes API:

```bash
# List all nodes
kubectl get nodes

# Delete the uninstalled node
kubectl delete node <NODE_NAME>
```

## Step 7: Verify Complete Removal

```bash
# Check that no RKE2 binaries remain
which rke2
ls /usr/local/bin/rke2*

# Check that the service definitions are gone
systemctl list-unit-files | grep rke2

# Verify no RKE2 processes are running
ps aux | grep rke2
```

## Complete Uninstall Script

For convenience, here is a combined script you can run on any node:

```bash
#!/bin/bash
# complete-rke2-uninstall.sh
# Run as root or with sudo

set -e

echo "Stopping RKE2 services..."
systemctl stop rke2-server 2>/dev/null || true
systemctl stop rke2-agent 2>/dev/null || true

echo "Running RKE2 uninstall scripts..."
if [ -f /usr/local/bin/rke2-uninstall.sh ]; then
    /usr/local/bin/rke2-uninstall.sh
elif [ -f /usr/local/bin/rke2-agent-uninstall.sh ]; then
    /usr/local/bin/rke2-agent-uninstall.sh
fi

echo "Removing residual data directories..."
rm -rf /var/lib/rancher/rke2/
rm -rf /etc/rancher/rke2/
rm -rf /var/log/pods/

echo "RKE2 uninstall complete."
```

```bash
# Make it executable and run
chmod +x complete-rke2-uninstall.sh
sudo ./complete-rke2-uninstall.sh
```

## Troubleshooting

### Uninstall Script Not Found

If the uninstall script is missing, the installation may have been non-standard. Manually remove the files:

```bash
# Remove binaries
sudo rm -f /usr/local/bin/rke2
sudo rm -f /usr/local/bin/kubectl
sudo rm -f /usr/local/bin/crictl

# Remove systemd service files
sudo rm -f /etc/systemd/system/rke2-server.service
sudo rm -f /etc/systemd/system/rke2-agent.service
sudo systemctl daemon-reload
```

### Containerd Namespace Still Exists

```bash
# List containerd namespaces
sudo /var/lib/rancher/rke2/bin/ctr namespaces list

# Remove any k8s.io namespace artifacts
sudo /var/lib/rancher/rke2/bin/ctr -n k8s.io containers list
```

## Conclusion

Cleanly uninstalling RKE2 involves stopping services, running the provided uninstall scripts, removing residual data directories, and cleaning up network artifacts. Always drain nodes before uninstalling in production to avoid disruption. The built-in uninstall scripts handle most of the heavy lifting, making the process straightforward.
