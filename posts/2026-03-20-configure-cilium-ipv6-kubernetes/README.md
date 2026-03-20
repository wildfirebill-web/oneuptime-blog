# How to Configure Cilium for IPv6 in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cilium, IPv6, Kubernetes, CNI, eBPF, Dual-Stack, NetworkPolicy

Description: Configure Cilium CNI for IPv6 and dual-stack Kubernetes clusters, including IPAM, network policies, and load balancing with eBPF.

## Introduction

Cilium is an eBPF-based CNI plugin for Kubernetes that provides high-performance networking, security, and observability. It supports IPv6, dual-stack, and IPv6-only clusters with native eBPF datapath acceleration.

## Step 1: Install Cilium with IPv6 Support

```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all \
    https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
tar xzvf cilium-linux-amd64.tar.gz
mv cilium /usr/local/bin

# Install Cilium with dual-stack enabled
cilium install \
    --set ipv6.enabled=true \
    --set ipam.mode=kubernetes \
    --set k8sServiceHost=2001:db8::controlplane \
    --set k8sServicePort=6443
```

## Step 2: Helm Configuration for Dual-Stack

```yaml
# cilium-values.yaml
ipv6:
  enabled: true

ipam:
  mode: "kubernetes"
  operator:
    clusterPoolIPv4PodCIDRList: ["10.0.0.0/8"]
    clusterPoolIPv6PodCIDRList: ["fd00:10::/104"]

# Enable dual-stack
enableIPv6Masquerade: false

# BPF-based IPv6 masquerade
bpf:
  masquerade: true

# kube-proxy replacement with IPv6
kubeProxyReplacement: true
k8sServiceHost: 2001:db8::controlplane
k8sServicePort: 6443

loadBalancer:
  mode: dsr     # Direct Server Return for IPv6
  algorithm: maglev
```

```bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
    --namespace kube-system \
    -f cilium-values.yaml
```

## Step 3: Verify IPv6 is Working

```bash
# Check Cilium status
cilium status

# Verify IPv6 addressing
kubectl get pods -o wide -n kube-system | grep cilium

# Check that pods get IPv6 addresses
kubectl get pods -o wide
# PODIP should show an IPv6 address (fd00:10::/104 range)

# Test IPv6 connectivity
kubectl exec -it <pod> -- ping6 -c 3 fd00:10::1

# Cilium connectivity test (includes IPv6)
cilium connectivity test
```

## Step 4: IPv6 Network Policy

```yaml
# netpolicy-ipv6.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-ipv6-ingress
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: web
  ingress:
    # Allow from IPv6 subnet
    - fromCIDR:
        - "2001:db8:client::/48"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP

    # Allow from pods in same namespace
    - fromEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: default
```

```bash
kubectl apply -f netpolicy-ipv6.yaml

# Verify policy
cilium policy get
```

## Step 5: Hubble Observability for IPv6

```bash
# Enable Hubble
cilium hubble enable

# Install Hubble CLI
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --fail -o hubble.tar.gz \
    "https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-amd64.tar.gz"
tar xzvf hubble.tar.gz
mv hubble /usr/local/bin

# Observe IPv6 flows
hubble observe --protocol ipv6 --follow

# Filter IPv6 traffic to specific pod
hubble observe \
    --pod default/web-pod \
    --ip-version ipv6 \
    --follow
```

## Step 6: IPv6-Only Cluster

```yaml
# cilium-ipv6-only.yaml
ipv4:
  enabled: false
ipv6:
  enabled: true

ipam:
  mode: kubernetes
  operator:
    clusterPoolIPv6PodCIDRList: ["fd00:10::/104"]

enableIPv6Masquerade: true
```

## Conclusion

Cilium's eBPF datapath provides native IPv6 support with sub-microsecond policy enforcement. Enable dual-stack with `ipv6.enabled=true` and configure IPv6 pod CIDR pools in IPAM settings. CiliumNetworkPolicy supports IPv6 CIDR-based rules. Use Hubble to observe IPv6 traffic flows and detect policy violations. Monitor Cilium agent health with OneUptime to alert on eBPF program load failures.
