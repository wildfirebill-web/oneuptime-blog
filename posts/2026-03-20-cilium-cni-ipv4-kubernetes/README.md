# How to Install Cilium CNI with IPv4 Networking in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cilium, Kubernetes, IPv4, CNI, eBPF, Networking

Description: Install Cilium CNI in a Kubernetes cluster using Helm or the Cilium CLI with IPv4 networking, eBPF acceleration, and observability features enabled.

Cilium is a CNI plugin powered by eBPF that provides high-performance networking, network policies, and deep observability. It can replace kube-proxy and supports advanced features like transparent encryption and Hubble monitoring.

## Prerequisites

```bash
# Cilium requires Linux kernel 4.9.17+ (5.10+ recommended for full feature set)
uname -r

# Initialize cluster WITHOUT kube-proxy (optional, Cilium can replace it)
sudo kubeadm init \
  --pod-network-cidr=10.0.0.0/16 \
  --skip-phases=addon/kube-proxy
# or with kube-proxy: just use standard kubeadm init
```

## Method 1: Install with Cilium CLI (Simplest)

```bash
# Install the Cilium CLI
curl -L --fail --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
sudo tar xzvf cilium-linux-amd64.tar.gz -C /usr/local/bin

# Install Cilium with default settings
cilium install

# With custom IPv4 CIDR
cilium install --set ipam.mode=cluster-pool \
  --set ipam.operator.clusterPoolIPv4PodCIDRList="10.0.0.0/16" \
  --set ipam.operator.clusterPoolIPv4MaskSize=24
```

## Method 2: Install with Helm

```bash
# Add the Cilium Helm repository
helm repo add cilium https://helm.cilium.io/
helm repo update

# Install Cilium with IPv4 configuration
helm install cilium cilium/cilium \
  --version 1.15.0 \
  --namespace kube-system \
  --set ipam.mode=cluster-pool \
  --set ipam.operator.clusterPoolIPv4PodCIDRList="10.0.0.0/16" \
  --set ipam.operator.clusterPoolIPv4MaskSize=24 \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<CONTROL_PLANE_IP> \
  --set k8sServicePort=6443
```

## Verifying the Installation

```bash
# Check Cilium status
cilium status

# Expected:
#     /¯¯\
#  /¯¯\__/¯¯\    Cilium:         OK
#  \__/¯¯\__/    Operator:       OK
#  /¯¯\__/¯¯\    Hubble Relay:   disabled
#  \__/¯¯\__/    ClusterMesh:    disabled

# View Cilium pods
kubectl get pods -n kube-system -l k8s-app=cilium

# Run the Cilium connectivity test
cilium connectivity test
```

## Enabling Hubble for Observability

```bash
# Enable Hubble (network observability)
cilium hubble enable

# Install Hubble CLI
curl -L --fail --remote-name-all \
  https://github.com/cilium/hubble/releases/latest/download/hubble-linux-amd64.tar.gz
sudo tar xzvf hubble-linux-amd64.tar.gz -C /usr/local/bin

# Port-forward to access Hubble
cilium hubble port-forward &

# Observe live network flows
hubble observe --follow
```

## Verifying IPv4 Pod Addresses

```bash
# Deploy a test pod and verify it gets a Cilium-assigned IPv4
kubectl run cilium-test --image=alpine --restart=Never -- sleep 3600
kubectl get pod cilium-test -o wide
# Pod should have IP in the 10.0.0.0/16 range

# Check Cilium endpoint info
kubectl exec -n kube-system $(kubectl get pods -n kube-system -l k8s-app=cilium -o name | head -1) \
  -- cilium endpoint list
```

## Checking Cilium IP Pool

```bash
# View CiliumNode objects showing IP allocation per node
kubectl get ciliumnodes -o yaml | grep -A10 ipam
```

Cilium's eBPF-based dataplane provides significantly better performance than iptables-based solutions, making it the preferred CNI for high-performance production clusters.
