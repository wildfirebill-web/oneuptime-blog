# How to Uninstall K3s Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Uninstall, DevOps, Linux

Description: A complete guide to cleanly uninstalling the K3s server, removing all associated services, binaries, and data from your system.

## Introduction

Whether you're decommissioning a cluster, migrating to a different Kubernetes distribution, or simply cleaning up a test environment, uninstalling K3s completely requires more than just deleting a binary. K3s installs systemd services, iptables rules, network interfaces, and data directories. This guide ensures a complete and clean removal.

## Prerequisites

- Root/sudo access to the K3s server node
- All workloads have been migrated or are no longer needed
- Agent nodes have been uninstalled first (recommended)

## Step 1: Drain and Remove Nodes (Pre-Uninstall)

Before uninstalling the server, gracefully remove workloads:

```bash
# From the server, drain all nodes

kubectl get nodes

# Drain each worker node first
kubectl drain <agent-node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force

# Delete nodes from the cluster
kubectl delete node <agent-node-name>

# Drain the server node itself (if multi-node)
kubectl drain <server-node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force
```

## Step 2: Use the Built-in Uninstall Script

K3s ships with an uninstall script that handles most of the cleanup:

```bash
# The uninstall script is placed at /usr/local/bin/k3s-uninstall.sh
# during installation

# Run the server uninstall script
/usr/local/bin/k3s-uninstall.sh
```

This script performs the following actions:
- Stops and disables the `k3s` systemd service
- Removes the K3s binary
- Removes the systemd service files
- Cleans up iptables rules
- Removes network interfaces (flannel, cni0)
- Deletes K3s data directories

## Step 3: Verify Service Removal

Confirm the K3s service is gone:

```bash
# Should show no k3s service
systemctl list-units | grep k3s

# Verify no k3s processes are running
ps aux | grep k3s

# Check if binary was removed
which k3s || echo "k3s binary removed"
ls -la /usr/local/bin/k3s 2>/dev/null || echo "Binary not found"
```

## Step 4: Manual Cleanup (If Script Is Missing)

If the uninstall script is not available, perform manual cleanup:

```bash
# Stop and disable K3s service
systemctl stop k3s
systemctl disable k3s

# Remove systemd service files
rm -f /etc/systemd/system/k3s.service
rm -f /etc/systemd/system/k3s.service.env
systemctl daemon-reload

# Remove the K3s binary
rm -f /usr/local/bin/k3s

# Remove kubectl, crictl, ctr symlinks
rm -f /usr/local/bin/kubectl
rm -f /usr/local/bin/crictl
rm -f /usr/local/bin/ctr

# Remove the uninstall scripts
rm -f /usr/local/bin/k3s-uninstall.sh
rm -f /usr/local/bin/k3s-agent-uninstall.sh
```

## Step 5: Remove Data Directories

Remove all K3s data:

```bash
# Main K3s data directory (contains etcd/SQLite, certs, manifests)
rm -rf /var/lib/rancher/k3s

# K3s configuration directory
rm -rf /etc/rancher/k3s

# Runtime directory
rm -rf /run/k3s

# Kubelet data
rm -rf /var/lib/kubelet

# Container runtime data
rm -rf /var/lib/containerd

# CNI configuration
rm -rf /etc/cni/net.d
rm -rf /var/lib/cni
rm -rf /opt/cni/bin
```

## Step 6: Remove Network Interfaces and iptables Rules

K3s creates network interfaces and iptables rules that persist after service removal:

```bash
# Remove flannel network interface
ip link delete flannel.1 2>/dev/null || true
ip link delete flannel-wg 2>/dev/null || true
ip link delete cni0 2>/dev/null || true

# Remove all K3s-related iptables rules
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X

# Restore default policies
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

# For IPv6
ip6tables -F
ip6tables -t nat -F
```

## Step 7: Remove kubeconfig

Clean up kubeconfig entries:

```bash
# Remove K3s kubeconfig
rm -f /etc/rancher/k3s/k3s.yaml

# Remove from user's kubeconfig if merged
kubectl config delete-cluster k3s-cluster 2>/dev/null || true
kubectl config delete-context k3s-context 2>/dev/null || true
```

## Step 8: Reboot (Recommended)

A reboot ensures all residual processes and kernel state are cleared:

```bash
# Reboot the server
reboot

# After reboot, verify clean state
# No k3s processes should be running
ps aux | grep -v grep | grep k3s || echo "No K3s processes found"
```

## Verify Complete Removal

After rebooting, confirm K3s is fully removed:

```bash
# Check for K3s binary
which k3s 2>/dev/null || echo "k3s not found (expected)"

# Check for systemd service
systemctl status k3s 2>/dev/null || echo "k3s service not found (expected)"

# Check for data directories
ls /var/lib/rancher 2>/dev/null || echo "Data directory removed (expected)"

# Check for network interfaces
ip link show | grep -E "flannel|cni0" || echo "No K3s interfaces (expected)"
```

## Conclusion

K3s provides a convenient uninstall script (`k3s-uninstall.sh`) that handles the majority of cleanup tasks. For complete removal, additionally clean up network interfaces, iptables rules, and data directories. A system reboot after uninstallation ensures a completely clean state, especially important if you plan to reinstall K3s or a different Kubernetes distribution on the same machine.
