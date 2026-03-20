# How to Configure K3s Cluster CIDR

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Rancher, Networking, CIDR, Pod Network

Description: Learn how to configure the K3s cluster CIDR (pod network range) to match your network topology and avoid IP address conflicts.

## Introduction

The cluster CIDR defines the IP address range used for pod networking in K3s. Every pod in the cluster receives an IP address from this range. The default is `10.42.0.0/16`, which provides over 65,000 pod IP addresses. However, you may need to change this if the default CIDR overlaps with your existing network infrastructure, if you need a larger or smaller address space, or if your security policies require specific IP ranges.

## Default K3s CIDR Values

| Setting | Default Value | Description |
|---------|--------------|-------------|
| `cluster-cidr` | `10.42.0.0/16` | IP range for pods |
| `service-cidr` | `10.43.0.0/16` | IP range for Services |
| `cluster-dns` | `10.43.0.10` | CoreDNS cluster IP |

## Important Constraints

1. The cluster CIDR must NOT overlap with your physical network ranges
2. The cluster CIDR and service CIDR must NOT overlap with each other
3. The cluster DNS IP must be within the service CIDR range
4. **Cannot be changed after cluster initialization** without rebuilding the cluster

## Checking for CIDR Conflicts

Before setting the cluster CIDR, verify it doesn't conflict with your network:

```bash
# Check your current network routes
ip route show

# Check all network interfaces and their subnets
ip addr show

# Common ranges to avoid:
# 10.0.0.0/8    - Common corporate networks
# 172.16.0.0/12 - Docker default bridge
# 192.168.0.0/16 - Home/office networks
```

## Setting a Custom Cluster CIDR

### Before Installation (Required)

The cluster CIDR must be set before the first server starts. It cannot be changed afterward.

```bash
sudo mkdir -p /etc/rancher/k3s

sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
token: "ClusterToken"

# Custom cluster CIDR for pod IPs
cluster-cidr: "172.20.0.0/16"

# Service CIDR must be different from cluster CIDR
service-cidr: "172.21.0.0/16"

# CoreDNS IP must be within service-cidr
cluster-dns: "172.21.0.10"
EOF

# Install K3s
curl -sfL https://get.k3s.io | sudo sh -

# Verify the CIDR was applied
kubectl cluster-info
kubectl get nodes -o yaml | grep podCIDR
```

## CIDR Sizing Guide

Choose the right CIDR size based on your expected pod count:

| CIDR | Addresses | Node CIDRs (/24 each) | Max Nodes |
|------|-----------|----------------------|-----------|
| /16 | 65,536 | 256 | ~256 (with 256 pods/node) |
| /17 | 32,768 | 128 | ~128 |
| /20 | 4,096 | 16 | ~16 |
| /24 | 256 | 1 | 1 (single node) |

```bash
# For a large cluster (1000+ nodes)
cluster-cidr: "10.42.0.0/12"   # 1 million IP addresses

# For a small cluster (< 32 nodes)
cluster-cidr: "10.42.0.0/20"   # 4096 IP addresses
```

## Common CIDR Configurations by Environment

### Home Lab

```yaml
# Avoid conflict with common home router range (192.168.0.0/16)
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
cluster-dns: "10.43.0.10"
```

### Corporate Environment (When 10.x.x.x is Already Used)

```yaml
# Use 172.16.x.x range (avoiding Docker's default 172.17.0.0/16)
cluster-cidr: "172.20.0.0/16"
service-cidr: "172.21.0.0/16"
cluster-dns: "172.21.0.10"
```

### Multi-Cluster Environment

For multiple K3s clusters on the same infrastructure, use non-overlapping ranges:

```yaml
# Cluster 1
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
cluster-dns: "10.43.0.10"
```

```yaml
# Cluster 2
cluster-cidr: "10.44.0.0/16"
service-cidr: "10.45.0.0/16"
cluster-dns: "10.45.0.10"
```

```yaml
# Cluster 3
cluster-cidr: "10.46.0.0/16"
service-cidr: "10.47.0.0/16"
cluster-dns: "10.47.0.10"
```

## Configuring IPv6 Dual-Stack

K3s supports dual-stack (IPv4 + IPv6) configurations:

```yaml
# /etc/rancher/k3s/config.yaml
cluster-cidr: "10.42.0.0/16,fd42::/48"
service-cidr: "10.43.0.0/16,fd43::/112"
cluster-dns: "10.43.0.10"
```

```bash
# Verify dual-stack is enabled
kubectl get nodes -o yaml | grep podCIDR
# Should show both IPv4 and IPv6 CIDRs
```

## Verifying CIDR Configuration

```bash
# After installation, verify the cluster CIDR
kubectl cluster-info dump | grep -m 5 "cluster-cidr"

# Or check the kube-controller-manager flags
ps aux | grep controller | tr ' ' '\n' | grep cluster-cidr

# Check pod IPs are within the expected range
kubectl get pods --all-namespaces -o wide | awk '{print $7}' | sort -u

# Check that CoreDNS has the correct cluster IP
kubectl -n kube-system get svc kube-dns
```

## Changing the CIDR After Installation

The cluster CIDR **cannot be changed** on a running cluster without rebuilding. If you need to change it:

```bash
# Step 1: Export all workloads
kubectl get all --all-namespaces -o yaml > workloads-backup.yaml

# Step 2: Uninstall K3s
sudo /usr/local/bin/k3s-uninstall.sh

# Step 3: Clean up network interfaces
sudo ip link delete flannel.1 2>/dev/null || true
sudo ip link delete cni0 2>/dev/null || true

# Step 4: Remove K3s data
sudo rm -rf /var/lib/rancher/k3s/

# Step 5: Reinstall with new CIDR
sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
cluster-cidr: "172.20.0.0/16"
service-cidr: "172.21.0.0/16"
cluster-dns: "172.21.0.10"
token: "ClusterToken"
EOF

curl -sfL https://get.k3s.io | sudo sh -

# Step 6: Re-apply workloads
kubectl apply -f workloads-backup.yaml
```

## Conclusion

Configuring the K3s cluster CIDR requires careful planning before installation to avoid conflicts with your physical network. The default `10.42.0.0/16` works well for most environments but must be changed if it overlaps with existing infrastructure. Always document your CIDR choices and ensure they don't conflict across multiple clusters in the same environment. Since the cluster CIDR cannot be changed post-installation, getting this right from the start is critical.
