# How to Uninstall K3s Agent

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kubernetes, Uninstall, DevOps, Linux

Description: Step-by-step instructions for completely removing the K3s agent from a worker node, including cleanup of services, binaries, and network configuration.

## Introduction

Removing a K3s agent (worker node) from your cluster requires both deregistering it from the Kubernetes control plane and cleaning up all local resources. This guide covers the complete process for cleanly removing a K3s agent, whether you're decommissioning a node or replacing it with a new one.

## Prerequisites

- Access to the K3s server (for kubectl operations)
- Root/sudo access to the agent node being removed
- The agent node name in Kubernetes

## Step 1: Drain the Agent Node from the Server

Before removing the agent, gracefully migrate workloads off the node:

```bash
# From the K3s server, identify the node name

kubectl get nodes

# Cordon the node to prevent new pods from being scheduled
kubectl cordon <agent-node-name>

# Drain the node - evict all running pods
kubectl drain <agent-node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --timeout=120s

# Verify no user workloads remain on the node
kubectl get pods -A --field-selector spec.nodeName=<agent-node-name>
```

## Step 2: Delete the Node from Kubernetes

Remove the node object from the cluster:

```bash
# Delete the node from Kubernetes
kubectl delete node <agent-node-name>

# Verify the node is removed
kubectl get nodes
```

## Step 3: Run the K3s Agent Uninstall Script

K3s installs an agent-specific uninstall script on agent nodes:

```bash
# SSH to the agent node
ssh user@<agent-node-ip>

# Run the agent uninstall script
/usr/local/bin/k3s-agent-uninstall.sh
```

The script automatically:
- Stops the `k3s-agent` systemd service
- Disables the service from auto-starting
- Removes the K3s binary and symlinks
- Cleans up systemd service files
- Removes network interfaces
- Cleans up iptables rules

## Step 4: Manual Cleanup (If Script Is Missing)

If the uninstall script is unavailable, perform manual cleanup:

```bash
# Stop and disable the agent service
systemctl stop k3s-agent
systemctl disable k3s-agent

# Remove service files
rm -f /etc/systemd/system/k3s-agent.service
rm -f /etc/systemd/system/k3s-agent.service.env
systemctl daemon-reload
systemctl reset-failed

# Remove K3s binary and symlinks
rm -f /usr/local/bin/k3s
rm -f /usr/local/bin/kubectl
rm -f /usr/local/bin/crictl
rm -f /usr/local/bin/ctr

# Remove uninstall scripts
rm -f /usr/local/bin/k3s-agent-uninstall.sh
rm -f /usr/local/bin/k3s-uninstall.sh
```

## Step 5: Remove Data Directories

Clean up K3s agent data:

```bash
# K3s runtime and agent data
rm -rf /var/lib/rancher/k3s

# K3s configuration
rm -rf /etc/rancher/k3s

# Runtime socket directory
rm -rf /run/k3s

# Kubelet data (pod volumes, certs, etc.)
rm -rf /var/lib/kubelet

# Container images and runtime data
rm -rf /var/lib/containerd

# CNI network plugins and config
rm -rf /etc/cni/net.d
rm -rf /var/lib/cni
rm -rf /opt/cni/bin
```

## Step 6: Clean Up Network Interfaces

Remove network interfaces created by K3s:

```bash
# Remove flannel interface
ip link delete flannel.1 2>/dev/null || echo "flannel.1 not found"
ip link delete flannel-wg 2>/dev/null || echo "flannel-wg not found"

# Remove CNI bridge interface
ip link delete cni0 2>/dev/null || echo "cni0 not found"

# Remove veth pairs (usually auto-cleaned, but just in case)
for iface in $(ip link show | grep -o 'veth[a-f0-9]*'); do
  ip link delete "$iface" 2>/dev/null || true
done

# Verify cleanup
ip link show | grep -E "flannel|cni0" || echo "K3s interfaces removed"
```

## Step 7: Clean Up iptables Rules

Remove K3s-created iptables rules:

```bash
# Flush all chains
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X

# Reset default policies to ACCEPT
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

# Save the clean rules
iptables-save > /etc/iptables/rules.v4
```

## Step 8: Remove Host Entries (If Applicable)

If the agent node was added to `/etc/hosts` on other nodes:

```bash
# On the K3s server and other nodes, remove the agent entry
sed -i '/<agent-node-ip>/d' /etc/hosts
```

## Step 9: Verify Clean Removal

After cleanup, verify the agent is completely removed:

```bash
# No k3s processes
ps aux | grep -v grep | grep k3s || echo "No K3s processes (expected)"

# No k3s binary
which k3s 2>/dev/null || echo "k3s binary not found (expected)"

# No k3s service
systemctl status k3s-agent 2>/dev/null || echo "k3s-agent service not found (expected)"

# No K3s data directories
ls /var/lib/rancher 2>/dev/null || echo "No K3s data directories (expected)"

# No K3s network interfaces
ip link show | grep -E "flannel|cni0" || echo "No K3s network interfaces (expected)"
```

## Reboot (Recommended)

A system reboot ensures all kernel state is cleared:

```bash
reboot
```

## Conclusion

Cleanly removing a K3s agent involves draining workloads from the Kubernetes side, then removing services, binaries, data directories, and network configuration on the node. Always drain the node before removal to avoid disrupting running workloads. The built-in `k3s-agent-uninstall.sh` script handles most of the cleanup automatically, with manual steps needed for any residual network configuration.
