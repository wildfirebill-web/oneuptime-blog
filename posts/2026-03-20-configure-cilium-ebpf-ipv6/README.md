# How to Configure Cilium eBPF-Based IPv6 Networking

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cilium, eBPF, IPv6, Kubernetes, CNI, XDP

Description: Configure Cilium's eBPF datapath for IPv6 in Kubernetes, including XDP acceleration, kube-proxy replacement, and eBPF-based IPv6 load balancing.

## Introduction

Cilium uses eBPF programs attached to network interfaces to implement the Kubernetes datapath. For IPv6, Cilium's eBPF programs handle pod-to-pod routing, service load balancing, and network policies entirely in the kernel, bypassing iptables and kube-proxy.

## Installing Cilium with eBPF and IPv6

```bash
# Install Cilium with full eBPF mode and IPv6

helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set ipv6.enabled=true \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=api.example.com \
  --set k8sServicePort=6443 \
  --set ipv6NativeRoutingCIDR="fd00:10::/104" \
  --set clusterPoolIPv6PodCIDRList[0]="fd00:10::/104" \
  --set bpf.masquerade=true \
  --set enableIPv6Masquerade=false \
  --set autoDirectNodeRoutes=true

# Verify eBPF programs loaded
cilium status --verbose
cilium bpf endpoint list
```

## Cilium eBPF Datapath Internals

```bash
# Show eBPF programs loaded on a node
bpftool prog list | grep cilium

# Show eBPF maps related to IPv6
bpftool map list | grep -E "cilium|6"

# Inspect the IPv6 neighbor table (eBPF map)
cilium bpf ipv6 list

# Show per-endpoint eBPF program attachment
# Each pod gets a tc ingress/egress program on its veth
ip link show type veth | grep lxc
bpftool net show dev lxc1234abcd

# Trace eBPF execution for debugging
cilium monitor --type trace --from-ip "fd00:10::1" --to-ip "fd00:10::2"
```

## XDP Acceleration for IPv6

```bash
# Enable XDP acceleration (requires supported NIC)
# XDP runs before the kernel network stack - fastest possible path

helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set loadBalancer.acceleration=native  # XDP native mode

# Verify XDP is enabled
cilium status | grep XDP
# Should show: XDP Acceleration:  NATIVE

# Check XDP program on physical interface
bpftool net show dev eth0
# xdp: id X  name bpf_xdp_entry
```

## IPv6 Kube-Proxy Replacement

```bash
# Verify kube-proxy replacement is active
cilium status | grep KubeProxyReplacement
# KubeProxyReplacement: True

# Check IPv6 service entries in eBPF
cilium service list
# Shows ClusterIP, NodePort, LoadBalancer services
# Each IPv6 service has eBPF LB rules

# Inspect a specific IPv6 service
kubectl get svc my-service -o jsonpath='{.spec.clusterIPs}'
# e.g.: ["10.96.0.10", "fd00:10:96::10"]

cilium bpf lb list | grep "fd00:10:96"
```

## eBPF-Based Network Policy for IPv6

```yaml
# IPv6 CiliumNetworkPolicy
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-ipv6-frontend-to-backend
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
    - fromCIDR:
        - "fd00:10::/104"  # Allow all pod traffic
```

```bash
# Verify policy is enforced via eBPF
cilium endpoint list | grep -E "backend|frontend"
cilium bpf policy get <endpoint-id>
# Shows eBPF policy map entries
```

## Monitoring Cilium IPv6 Performance

```bash
# Hubble flow monitoring for IPv6
hubble observe --type l3-l4 --ip-version ipv6

# Per-endpoint IPv6 packet counts
cilium bpf endpoint list | head -20

# Network policy drop counters
cilium bpf policy get <endpoint-id> | grep "denied"

# Hubble metrics for Prometheus
# cilium_drop_count_total{reason="POLICY_DENIED", direction="INGRESS"} 42
```

## Conclusion

Cilium's eBPF datapath provides the most efficient IPv6 networking for Kubernetes, with optional XDP acceleration for line-rate performance. kube-proxy replacement eliminates iptables overhead for IPv6 service routing. Use `cilium monitor` and Hubble for real-time flow visibility. Monitor pod connectivity and network policy effectiveness with OneUptime synthetic checks.
