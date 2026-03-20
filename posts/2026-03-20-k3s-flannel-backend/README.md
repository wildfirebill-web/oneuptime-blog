# How to Configure K3s Flannel Backend

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Rancher, Flannel, Networking, CNI

Description: Learn how to configure different Flannel backend modes in K3s including VXLAN, host-gw, WireGuard, and IPsec for optimal networking performance.

## Introduction

K3s uses Flannel as its default CNI plugin. Flannel supports multiple backend modes that determine how pod network traffic is encapsulated and routed between nodes. Choosing the right backend for your environment can significantly impact network performance, security, and compatibility. This guide covers all available Flannel backends and when to use each.

## Available Flannel Backends

| Backend | Encryption | Performance | Requirements |
|---------|-----------|-------------|-------------|
| `vxlan` (default) | No | Good | None |
| `host-gw` | No | Excellent | L2 network |
| `wireguard-native` | Yes | Very Good | Kernel 5.6+ |
| `ipsec` | Yes | Good | Strongswan |
| `none` | N/A | N/A | Custom CNI |

## Configuring the Flannel Backend

### VXLAN (Default)

VXLAN is the default and most compatible backend. It encapsulates pod traffic in UDP packets:

```yaml
# /etc/rancher/k3s/config.yaml

flannel-backend: "vxlan"
```

```bash
# Install K3s with explicit VXLAN backend
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_EXEC="--flannel-backend=vxlan" \
    sudo sh -

# Verify VXLAN interface was created
ip link show flannel.1
```

**When to use VXLAN:**
- Most environments (cloud, VMs, bare metal)
- When nodes are on different L2 network segments
- When you don't need encryption

### host-gw (Highest Performance)

`host-gw` avoids encapsulation overhead by using the host's routing table:

```yaml
flannel-backend: "host-gw"
```

```bash
# Install with host-gw backend
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_EXEC="--flannel-backend=host-gw" \
    sudo sh -

# Verify routes were added
ip route show | grep "via"
# You should see routes like: 10.42.X.0/24 via NODE_IP
```

**Requirements for host-gw:**
- All nodes must be on the **same L2 broadcast domain** (same subnet)
- Nodes must be able to route directly to each other without a router

**When to use host-gw:**
- Bare metal clusters where all nodes are on the same switch
- Single-site clusters where maximum throughput is needed
- When latency matters (no encapsulation = lower latency)

### WireGuard (Encrypted, Recommended for Security)

WireGuard provides modern, efficient encryption:

```yaml
# Requires kernel 5.6+ (Ubuntu 20.04+, Fedora 32+, RHEL 8.3+)
flannel-backend: "wireguard-native"
```

```bash
# Check WireGuard support
modinfo wireguard 2>/dev/null && echo "WireGuard supported" || echo "WireGuard not available"

# On Ubuntu, install WireGuard if not built into kernel
sudo apt-get install -y wireguard

# Install K3s with WireGuard
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_EXEC="--flannel-backend=wireguard-native" \
    sudo sh -

# Verify WireGuard interface
sudo wg show
ip link show flannel-wg
```

**When to use WireGuard:**
- Clusters spanning multiple sites or data centers
- Public cloud clusters where node traffic crosses the internet
- Multi-cloud clusters
- Any environment requiring encrypted pod traffic

### IPsec (Legacy Encryption)

IPsec provides strong encryption but requires strongSwan:

```bash
# Install strongSwan on all nodes
sudo apt-get install -y strongswan

# Configure K3s with IPsec backend
sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
flannel-backend: "ipsec"
EOF

curl -sfL https://get.k3s.io | sudo sh -
```

**When to use IPsec:**
- Compliance requirements mandating IPsec specifically
- Integration with existing IPsec infrastructure

### none (Bring Your Own CNI)

Disable Flannel entirely and install your own CNI:

```yaml
flannel-backend: "none"
```

```bash
# Install K3s without CNI
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_EXEC="--flannel-backend=none --disable-network-policy" \
    sudo sh -

# After K3s starts, install your CNI manually
# Example: Cilium
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
    --namespace kube-system \
    --set operator.replicas=1
```

## Switching Backends on an Existing Cluster

Changing the Flannel backend requires resetting the network:

```bash
# Stop K3s
sudo systemctl stop k3s

# Remove the Flannel database
sudo rm -f /var/lib/rancher/k3s/agent/etc/cni/net.d/10-flannel.conflist
sudo rm -rf /var/lib/rancher/k3s/agent/etc/cni/net.d/flannel.d/

# Remove old Flannel interfaces
sudo ip link delete flannel.1 2>/dev/null || true
sudo ip link delete flannel-wg 2>/dev/null || true

# Update config.yaml
sudo sed -i 's/flannel-backend:.*/flannel-backend: "wireguard-native"/' /etc/rancher/k3s/config.yaml

# Start K3s
sudo systemctl start k3s

# Verify new interface
sudo journalctl -u k3s | grep -i flannel
```

## Performance Comparison

```bash
# Install iperf3 for network benchmarking
sudo apt-get install -y iperf3

# Run iperf3 server on one pod
kubectl run iperf-server --image=networkstatic/iperf3 --restart=Never -- iperf3 -s

# Run iperf3 client from another pod
IPERF_SERVER_IP=$(kubectl get pod iperf-server -o jsonpath='{.status.podIP}')
kubectl run iperf-client --image=networkstatic/iperf3 --restart=Never -- \
    iperf3 -c $IPERF_SERVER_IP -t 10

# View results
kubectl logs iperf-client
```

## Flannel VXLAN MTU Configuration

VXLAN encapsulation reduces the effective MTU by 50 bytes:

```yaml
# For standard 1500 MTU physical NICs
# VXLAN MTU should be 1450 (1500 - 50 overhead)

# K3s sets this automatically, but you can configure it via:
flannel-backend: "vxlan"
kube-proxy-arg:
  - "v=4"  # Increase verbosity to see MTU in use
```

Check the effective MTU:

```bash
# Check the flannel.1 interface MTU
ip link show flannel.1 | grep mtu

# Check pod network MTU
kubectl run mtu-check --image=busybox --restart=Never -- \
    sh -c "cat /sys/class/net/eth0/mtu"
kubectl logs mtu-check
```

## Conclusion

K3s's Flannel backend flexibility allows you to optimize the network for your specific environment. VXLAN is the safe default for cloud and multi-segment networks. Use `host-gw` for maximum performance when all nodes are on the same L2 network. Choose `wireguard-native` when you need encryption with modern performance - it's the best choice for multi-site or cloud deployments. For organizations with strict compliance requirements, `ipsec` is available. If you need a CNI feature not available in Flannel (like advanced network policies), use `none` and install Calico or Cilium.
