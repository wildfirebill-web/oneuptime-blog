# How to Configure Kubernetes Dual-Stack Networking with IPv4 and IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, Dual Stack, IPv4, IPv6, Networking, CNI

Description: Enable Kubernetes dual-stack networking to assign both IPv4 and IPv6 addresses to pods and services, allowing gradual IPv6 adoption.

Kubernetes dual-stack networking allows pods and services to simultaneously have both IPv4 and IPv6 addresses. This is essential for IPv6 readiness while maintaining IPv4 compatibility.

## Prerequisites

```bash
# Requires Kubernetes 1.21+ (stable dual-stack)

kubectl version

# The CNI plugin must support dual-stack (Calico, Cilium, Flannel with vxlan)
# Nodes must have both IPv4 and IPv6 addresses
ip addr show
```

## Initializing a Dual-Stack Cluster with kubeadm

```yaml
# kubeadm-dualstack.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  # IPv4 CIDR comes first (IPv4 is the primary stack)
  podSubnet: "10.244.0.0/16,fd00:10:244::/48"
  serviceSubnet: "10.96.0.0/12,fd00:10:96::/108"
```

```bash
sudo kubeadm init --config kubeadm-dualstack.yaml
```

## Installing Calico for Dual-Stack

```yaml
# calico-dualstack.yaml - IPAddressPool for both families
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    # IPv4 pool
    - name: ipv4-pool
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
    # IPv6 pool
    - name: ipv6-pool
      cidr: fd00:10:244::/48
      encapsulation: None
      natOutgoing: Enabled
```

## Verifying Dual-Stack Node Configuration

```bash
# Nodes should show both IPv4 and IPv6 addresses
kubectl get nodes -o wide
# INTERNAL-IP: 192.168.1.10,fd00::1  (both stacks)

# Check controller-manager flags
kubectl get pod -n kube-system kube-controller-manager-<node> -o yaml | \
  grep cluster-cidr
# Expected: --cluster-cidr=10.244.0.0/16,fd00:10:244::/48
```

## Creating a Dual-Stack Service

```yaml
# dual-stack-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-dual-stack-service
spec:
  selector:
    app: my-app
  # Prefer IPv4 for traffic (IPv4 is primary)
  ipFamilyPolicy: PreferDualStack
  # Or require both stacks:
  # ipFamilyPolicy: RequireDualStack
  ipFamilies:
  - IPv4
  - IPv6
  ports:
  - port: 80
    targetPort: 8080
```

```bash
kubectl apply -f dual-stack-service.yaml

# View the service's dual-stack ClusterIPs
kubectl get svc my-dual-stack-service -o yaml | grep -A5 clusterIPs
# clusterIPs:
# - 10.96.45.123      (IPv4)
# - fd00:10:96::abc   (IPv6)
```

## Testing Dual-Stack Pod Addresses

```bash
# Deploy a pod and check both IP addresses
kubectl run dualstack-test --image=alpine --restart=Never -- sleep 3600
kubectl get pod dualstack-test -o yaml | grep -A5 podIPs
# podIPs:
# - ip: 10.244.1.5     (IPv4)
# - ip: fd00:10:244::5 (IPv6)

# Test connectivity both ways
kubectl exec dualstack-test -- ping -c 2 10.96.45.123
kubectl exec dualstack-test -- ping6 -c 2 fd00:10:96::abc
```

## ipFamilyPolicy Options

| Policy | Behavior |
|---|---|
| `SingleStack` | Only one IP family (IPv4 or IPv6) |
| `PreferDualStack` | Dual-stack if available, single-stack fallback |
| `RequireDualStack` | Must have both; fails if cluster doesn't support it |

Dual-stack is the recommended path for new Kubernetes deployments in environments moving toward IPv6 adoption.
