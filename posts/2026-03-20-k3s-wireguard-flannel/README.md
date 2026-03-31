# How to Set Up WireGuard for K3s Flannel Backend

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kubernetes, WireGuard, Flannel, Networking, Security, VPN

Description: Learn how to configure K3s Flannel to use WireGuard as its backend for encrypted pod-to-pod communication across nodes.

## Introduction

By default, K3s uses Flannel with VXLAN backend for pod networking, which provides no encryption for inter-node traffic. For security-sensitive deployments, K3s supports using **WireGuard** as the Flannel backend to encrypt all pod traffic between nodes. WireGuard offers excellent performance with minimal overhead compared to other VPN solutions. This guide covers configuring K3s with the WireGuard Flannel backend.

## Prerequisites

- Linux kernel 5.6+ (WireGuard is built-in) or WireGuard kernel module installed
- Root/sudo access on all K3s nodes
- K3s not yet installed (for fresh setup) or ability to restart the cluster

## Step 1: Install WireGuard Kernel Module

WireGuard support is required on all nodes:

```bash
# Check if WireGuard is available (kernel 5.6+)

modinfo wireguard 2>/dev/null && echo "WireGuard module available"

# Check kernel version
uname -r

# For older kernels, install WireGuard
# Ubuntu 18.04/20.04
apt-get update && apt-get install -y wireguard wireguard-tools

# RHEL/CentOS 8
dnf install -y elrepo-release
dnf install -y kmod-wireguard wireguard-tools

# Raspberry Pi (Raspbian)
apt-get install -y raspberrypi-kernel-headers
apt-get install -y wireguard

# Load the WireGuard kernel module
modprobe wireguard

# Verify the module is loaded
lsmod | grep wireguard

# Make it persistent
echo "wireguard" >> /etc/modules-load.d/wireguard.conf
```

## Step 2: Install K3s with WireGuard Backend

### Fresh Installation

```bash
# Install K3s server with WireGuard Flannel backend
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="--flannel-backend=wireguard-native" \
  sh -

# For WireGuard with kernel module (older WireGuard versions)
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="--flannel-backend=wireguard" \
  sh -
```

### Using Config File

```yaml
# /etc/rancher/k3s/config.yaml
flannel-backend: wireguard-native
```

Then install K3s:

```bash
curl -sfL https://get.k3s.io | sh -
```

## Step 3: Install K3s Agents with WireGuard

Agent nodes also need WireGuard installed:

```bash
# Ensure WireGuard is installed on agent nodes
apt-get install -y wireguard

# Install K3s agent
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<server-ip>:6443 \
  K3S_TOKEN=<node-token> \
  sh -

# The agent automatically uses WireGuard when the server uses it
```

## Step 4: Verify WireGuard Interfaces

After installation, verify WireGuard interfaces are created:

```bash
# List network interfaces
ip link show | grep flannel-wg
# or
ip link show | grep wireguard

# View WireGuard interface details
ip link show flannel-wg

# Check WireGuard status (requires wireguard-tools)
wg show

# Expected output:
# interface: flannel-wg
#   public key: <base64-public-key>
#   listening port: 51820
#
# peer: <node2-public-key>
#   endpoint: 192.168.1.11:51820
#   latest handshake: X seconds ago
#   transfer: X MiB received, X MiB sent
```

## Step 5: Verify Encrypted Pod Communication

```bash
# Deploy test pods on different nodes
kubectl run pod-a --image=busybox --restart=Never \
  --overrides='{"spec":{"nodeName":"node1"}}' \
  -- sleep 3600

kubectl run pod-b --image=busybox --restart=Never \
  --overrides='{"spec":{"nodeName":"node2"}}' \
  -- sleep 3600

# Get Pod A's IP
POD_A_IP=$(kubectl get pod pod-a -o jsonpath='{.status.podIP}')

# Ping from Pod B to Pod A (cross-node communication)
kubectl exec pod-b -- ping -c 4 $POD_A_IP

# Verify WireGuard is encrypting traffic
# On the host, you should see WireGuard traffic on UDP 51820
# but NOT plaintext pod traffic

# tcpdump to verify
tcpdump -i eth0 -n port 51820 | head -20
# Should show UDP traffic on 51820 (WireGuard encrypted)

# Clean up
kubectl delete pod pod-a pod-b
```

## Step 6: Configure WireGuard Port (Optional)

The default WireGuard port is 51820. If you need to change it:

```bash
# Check current K3s WireGuard port
ss -ulnp | grep 51820

# K3s doesn't directly expose WireGuard port configuration
# via config flags - the port is determined by Flannel

# Ensure port 51820/UDP is open in your firewall
# UFW
ufw allow 51820/udp

# firewalld
firewall-cmd --permanent --add-port=51820/udp
firewall-cmd --reload

# iptables
iptables -A INPUT -p udp --dport 51820 -j ACCEPT
```

## Step 7: WireGuard NXT (WireGuard Native with Native Routes)

K3s 1.26+ supports `wireguard-native` which uses the kernel's native WireGuard implementation for better performance:

```yaml
# /etc/rancher/k3s/config.yaml
flannel-backend: wireguard-native
# Enable native WireGuard
flannel-ipv6-masq: false  # If not using IPv6
```

```bash
# Verify native WireGuard is in use
wg show
# Interface name should be 'flannel-wg' or 'flannel-wg-v6' for IPv6
```

## Step 8: Monitor WireGuard Statistics

```bash
# View WireGuard statistics
wg show all dump

# Monitor transfer statistics
watch -n 2 'wg show all'

# Check K3s logs for WireGuard-related events
journalctl -u k3s | grep -i wireguard

# Check WireGuard peer connections
wg show all peers
```

## Troubleshooting

**WireGuard interface not created:**
```bash
# Check if WireGuard module is loaded
lsmod | grep wireguard
# If missing:
modprobe wireguard
```

**Pods can't communicate across nodes:**
```bash
# Verify WireGuard handshakes are happening
wg show
# 'latest handshake' should be recent

# Check if port 51820 is accessible between nodes
nc -uzv <other-node-ip> 51820
```

**Poor performance:**
```bash
# WireGuard performance should be excellent
# If slow, check CPU usage on the WireGuard cipher operations
# Consider using wireguard-native backend for better kernel integration
```

## Conclusion

Configuring K3s with WireGuard as the Flannel backend encrypts all inter-node pod traffic with minimal performance overhead. WireGuard's simplicity and excellent kernel integration make it ideal for K3s deployments where you need encrypted pod networking. For new clusters, use `wireguard-native` for the best performance; for existing clusters, plan a maintenance window to switch backends as it requires a restart of all K3s nodes.
