# How to Configure Calico eBPF Mode with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Calico, eBPF, IPv6, Kubernetes, CNI, XDP

Description: Configure Calico in eBPF mode for IPv6 Kubernetes workloads, replacing iptables and kube-proxy with eBPF programs for better performance and IPv6 support.

## Introduction

Calico eBPF mode uses the Linux eBPF subsystem instead of iptables for packet processing, providing lower latency and better performance. It supports IPv6 natively and can replace kube-proxy for IPv6 service routing.

## Prerequisites and Installation

```bash
# Requirements for eBPF mode:
# - Linux kernel >= 5.3
# - Calico >= 3.13
# - Kubernetes >= 1.16
# - kube-proxy NOT running (it will be replaced by Calico eBPF)

# First, install Calico with IPv6 enabled
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Or via Helm with IPv6
helm install calico projectcalico/tigera-operator \
  --namespace tigera-operator \
  --create-namespace
```

## Enable eBPF Mode

```bash
# 1. Disable kube-proxy (before enabling Calico eBPF)
kubectl patch ds kube-proxy -n kube-system \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'

# 2. Configure Calico to use eBPF
kubectl patch installation default --type merge \
  -p '{"spec": {"calicoNetwork": {"linuxDataplane": "BPF"}}}'

# Wait for rollout
kubectl rollout status ds/calico-node -n calico-system

# 3. Verify eBPF mode
kubectl exec -n calico-system ds/calico-node -- \
  calico-node -felix-live 2>/dev/null | grep "BPF"
```

## Calico eBPF with IPv6 IPPools

```yaml
# Dual-stack IPPools for eBPF mode
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ipv6-pool
spec:
  cidr: fd00:10:244::/48
  blockSize: 122   # /122 per node (64 IPs each)
  ipipMode: Never   # eBPF mode doesn't use IPIP
  vxlanMode: Never  # eBPF uses native routing
  natOutgoing: false
  nodeSelector: all()
```

```bash
# Apply the pool
kubectl apply -f ipv6-pool.yaml

# Verify
kubectl exec -n calico-system ds/calico-node -- \
  calicoctl get ippool ipv6-pool -o yaml
```

## kube-proxy Replacement in eBPF Mode

```bash
# Configure BPF kube-proxy replacement
kubectl patch configmap calico-config -n kube-system --type merge \
  -p '{"data": {"bpf-kube-proxy-iptables-cleanup-enabled": "true"}}'

# Set the API server address for kube-proxy replacement
kubectl patch felixconfiguration default --type merge -p '{
  "spec": {
    "bpfKubeProxyIptablesCleanupEnabled": true,
    "bpfLogLevel": "off"
  }
}'

# Verify services are handled by eBPF
kubectl exec -n calico-system ds/calico-node -- \
  calico-node -bpf-dump-maps service 2>/dev/null | head -20
```

## Troubleshooting eBPF Mode

```bash
# Check Felix eBPF status
kubectl logs -n calico-system ds/calico-node -c calico-node | \
  grep -i "bpf\|ebpf"

# List loaded eBPF programs
bpftool prog list | grep calico | head -20

# Check eBPF maps
bpftool map list | grep calico | head -20

# Verify IPv6 routing in eBPF
# On a node, check that pod IPv6 addresses are in eBPF route map
kubectl exec -n calico-system ds/calico-node -- \
  calico-node -bpf-dump-maps route 2>/dev/null | \
  grep "fd00:10:244"

# Felix logs for debugging
kubectl logs -n calico-system ds/calico-node -c calico-node --follow | \
  grep -iE "ipv6|bpf|error"
```

## Performance Comparison

```bash
# Benchmark eBPF vs iptables mode
# Pod to pod throughput with iperf3

# Test pod 1 (server)
kubectl exec server-pod -- iperf3 -s -6

# Test pod 2 (client)
kubectl exec client-pod -- iperf3 -c [fd00:10:244::server] -6 -t 30

# Expected improvement with eBPF:
# iptables mode: ~9 Gbps (10GbE)
# eBPF mode: ~9.8 Gbps (less overhead)
# XDP mode: line-rate improvement for NodePort
```

## Conclusion

Calico eBPF mode improves IPv6 performance by replacing iptables with efficient eBPF programs. Disable kube-proxy before enabling eBPF, and configure `/122` IPv6 block sizes for efficient IPAM. Use `calico-node -bpf-dump-maps` to inspect runtime state. Monitor node-to-node latency and pod connectivity with OneUptime to verify that eBPF mode is performing as expected.
