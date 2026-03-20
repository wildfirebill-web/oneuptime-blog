# How to Configure Pod CIDR for IPv6 in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, IPv6, Pod CIDR, Networking, Dual-Stack, CNI

Description: Configure IPv6 Pod CIDR ranges in Kubernetes, understand how pod CIDRs are allocated per node, and verify that pods receive IPv6 addresses from the configured ranges.

## Introduction

The pod CIDR in Kubernetes defines the IP address range from which pods receive their addresses. In dual-stack clusters, the pod CIDR includes both IPv4 and IPv6 ranges, typically specified as comma-separated values. Each node receives a slice of the cluster-wide pod CIDR — in dual-stack, each node gets one IPv4 and one IPv6 block. The CNI plugin uses these per-node CIDRs to assign addresses to pods.

## Configure Pod CIDR in kubeadm

```yaml
# kubeadm-config.yaml — dual-stack pod CIDR
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.29.0
networking:
  # IPv4 /16 = 65536 IPs, each node gets /24 (256 pods/node)
  # IPv6 /56 = 256 /64 subnets, each node gets a /64
  podSubnet: "10.244.0.0/16,fd00:10:244::/56"
  serviceSubnet: "10.96.0.0/12,fd00:10:96::/108"
```

```bash
# Initialize with these CIDRs
sudo kubeadm init --config kubeadm-config.yaml

# Or directly via flags
sudo kubeadm init \
    --pod-network-cidr="10.244.0.0/16,fd00:10:244::/56" \
    --service-cidr="10.96.0.0/12,fd00:10:96::/108"
```

## View Per-Node Pod CIDR Allocation

```bash
# Each node gets a portion of the cluster pod CIDR
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.podCIDRs}{"\n"}{end}'

# Example output:
# node1: ["10.244.0.0/24","fd00:10:244::/64"]
# node2: ["10.244.1.0/24","fd00:10:244:1::/64"]
# node3: ["10.244.2.0/24","fd00:10:244:2::/64"]

# Detailed node inspection
kubectl describe node node1 | grep -A5 "PodCIDR"
```

## Verify Pod IPv6 Address from CIDR

```bash
# Create a test pod and check IPv6 assignment
kubectl run testpod --image=nginx --restart=Never

# Wait for pod to be running
kubectl wait --for=condition=Ready pod/testpod --timeout=60s

# Check pod IPs
kubectl get pod testpod -o jsonpath='{.status.podIPs}'
# [{"ip":"10.244.0.5"},{"ip":"fd00:10:244::5"}]

# Verify IPv6 is from the node's pod CIDR
kubectl exec testpod -- ip -6 addr show eth0
# inet6 fd00:10:244::5/64 scope global

# Get all pod IPs across the cluster
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.name}: {.status.podIPs}{"\n"}{end}'
```

## Configure CNI for Pod CIDR

```yaml
# Calico dual-stack with pod CIDRs
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      # IPv4 pool matching pod CIDR
      - blockSize: 26
        cidr: 10.244.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
      # IPv6 pool matching pod CIDR
      - blockSize: 122
        cidr: fd00:10:244::/56
        encapsulation: VXLAN
        natOutgoing: Enabled
        nodeSelector: all()
```

```yaml
# Flannel dual-stack ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
data:
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "IPv6Network": "fd00:10:244::/56",
      "EnableIPv6": true,
      "Backend": {
        "Type": "vxlan"
      }
    }
```

## Expand Pod CIDR (Advanced)

```bash
# Pod CIDRs cannot be changed after cluster initialization
# If you need more addresses, you must recreate the cluster

# Check current pod CIDR
kubectl cluster-info dump | grep podCIDR

# Plan your CIDR sizing:
# /16 IPv4 = 65536 addresses, ~256 nodes at /24 each (254 pods/node)
# /56 IPv6 = 256 /64 subnets, one per node

# Recommended sizing for production:
# IPv4 pod CIDR: 10.244.0.0/14 (1M addresses)
# IPv6 pod CIDR: fd00::/56 (256 /64 node subnets)
```

## Conclusion

Configure dual-stack pod CIDRs in Kubernetes by providing comma-separated IPv4 and IPv6 CIDRs in `podSubnet` or `--pod-network-cidr`. Each node receives a slice of the cluster pod CIDR — visible in `node.spec.podCIDRs`. The CNI plugin (Calico, Cilium, Flannel) must be configured with matching IPv4 and IPv6 IP pools. Pods in dual-stack clusters automatically receive both IPv4 and IPv6 addresses within their node's CIDR slices. Size your pod CIDRs at cluster creation — they cannot be changed without recreating the cluster.
