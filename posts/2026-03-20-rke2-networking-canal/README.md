# How to Configure RKE2 Networking with Canal - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Canal, CNI, Networking, Rancher

Description: Learn how to configure Canal CNI (the default RKE2 network plugin combining Flannel and Calico) for Kubernetes pod networking.

Canal is the default CNI (Container Network Interface) plugin in RKE2. It combines Flannel for pod networking with Calico for network policy enforcement, giving you both simple overlay networking and powerful network policy capabilities. This guide covers Canal configuration options in RKE2.

## Prerequisites

- RKE2 installed or being installed
- Understanding of Kubernetes networking basics
- Network access between all nodes

## What is Canal CNI?

Canal combines two powerful networking solutions:

- **Flannel**: Provides a simple overlay network (VXLAN) for pod-to-pod communication
- **Calico**: Provides network policy enforcement and optional BGP routing

## Step 1: Configure Canal in RKE2

Canal is the default CNI, so minimal configuration is needed:

```yaml
# /etc/rancher/rke2/config.yaml

# Canal is the default - this is optional but explicit
cni: canal

# Pod network CIDR
cluster-cidr: 10.42.0.0/16

# Flannel backend options
# Options: vxlan (default), wireguard-native, host-gw, udp
# For Canal in RKE2, backend is configured separately
```

## Step 2: Configure Flannel Backend

```yaml
# /etc/rancher/rke2/config.yaml - Configure Flannel backend
cni: canal

# Configure Flannel to use WireGuard for encrypted pod communication
# This requires kernel 5.6+ with WireGuard support
flannel-backend: wireguard-native

# Or use VXLAN (default, works on most kernels)
# flannel-backend: vxlan

# Or use host-gw for non-overlay networking (all nodes must be on same L2)
# flannel-backend: host-gw
```

## Step 3: Configure Network Policies with Canal/Calico

Canal's Calico component provides Kubernetes network policy support:

```yaml
# Example NetworkPolicy using Calico-backed Canal
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
```

```bash
# Apply network policy
kubectl apply -f network-policy.yaml

# Verify network policy is enforced
kubectl get networkpolicy -n my-app

# Test connectivity (should be blocked without matching labels)
kubectl run test-pod --image=busybox -n my-app -- \
  wget -qO- --timeout=5 http://backend-service:8080
```

## Step 4: Using GlobalNetworkPolicy (Calico-specific)

When using Canal, you can use Calico's GlobalNetworkPolicy for cluster-wide rules:

```yaml
# global-network-policy.yaml - Requires Calico CRDs
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-all-except-dns
spec:
  # Apply to all pods in the cluster
  selector: all()
  types:
  - Ingress
  - Egress
  egress:
  # Allow DNS
  - action: Allow
    protocol: UDP
    destination:
      ports: [53]
  # Allow all within the cluster
  - action: Allow
    destination:
      selector: all()
  ingress:
  # Allow all within the cluster
  - action: Allow
    source:
      selector: all()
```

## Step 5: Configure Canal with Custom VXLAN Settings

```yaml
# HelmChartConfig to customize Canal/Flannel settings
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-canal
  namespace: kube-system
spec:
  valuesContent: |-
    flannel:
      # VXLAN port (default: 8472)
      # vxlanPort: 8472

      # VXLAN ID
      # vxlanID: 1

    calico:
      # Container interface name prefix
      containerInterface: "eth"

      # MTU for Canal network interfaces
      # Leave empty to auto-detect
      mtu: "0"

      # Enable IPv6 dual-stack
      # ipv6:
      #   enabled: false
```

## Step 6: Monitor Canal Network Health

```bash
# Check Canal pods in kube-system
kubectl get pods -n kube-system | grep canal

# Check the Canal DaemonSet
kubectl describe daemonset rke2-canal -n kube-system

# View Canal configuration
kubectl get configmap rke2-canal-config -n kube-system -o yaml

# Check Flannel logs
kubectl logs -n kube-system \
  $(kubectl get pods -n kube-system -l k8s-app=canal -o name | head -1) \
  -c kube-flannel --tail=50

# Check Calico logs
kubectl logs -n kube-system \
  $(kubectl get pods -n kube-system -l k8s-app=canal -o name | head -1) \
  -c calico-node --tail=50

# Test pod connectivity
kubectl run ping-test --image=busybox --rm -it -- ping 10.42.0.1
```

## Step 7: Troubleshoot Canal Networking Issues

```bash
# Check if VXLAN interfaces are created on nodes
ip link show | grep flannel

# Check the flannel subnet configuration
cat /run/flannel/subnet.env

# Check Canal's IP address management
kubectl get ippools -A 2>/dev/null || \
  echo "Calico IPAM pools (may need Calico CRDs installed separately)"

# Check node routing
ip route show | grep 10.42

# Verify pod IPs are in the expected range
kubectl get pods -A -o wide | awk '{print $7}' | sort | uniq
```

## Conclusion

Canal CNI provides RKE2 with a solid default networking solution that combines the simplicity of Flannel with the network policy capabilities of Calico. For most production deployments, the default Canal configuration works well. When you need network isolation between namespaces, Canal's network policy support via Calico provides the enforcement engine. For environments requiring encrypted pod-to-pod traffic, configuring the WireGuard backend provides native kernel-level encryption without the overhead of a service mesh.
