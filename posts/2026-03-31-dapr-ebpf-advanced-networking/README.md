# How to Use Dapr with eBPF for Advanced Networking

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, eBPF, Networking, Cilium, Observability, Kubernetes

Description: Learn how Dapr integrates with eBPF-based networking tools like Cilium to enhance observability, enforce network policies, and reduce sidecar overhead.

---

## Dapr and eBPF Networking

eBPF (extended Berkeley Packet Filter) enables kernel-level packet processing without modifying kernel code. When combined with Dapr, eBPF tools like Cilium provide network-level observability and policy enforcement that complements Dapr's application-level features.

## Why Use eBPF with Dapr

Dapr sidecars handle application-level concerns (service invocation, pub/sub, secrets). eBPF complements this by providing:

- **Network-level observability** - See all pod-to-pod communication at the kernel level
- **Network policies** - Drop traffic that bypasses the Dapr sidecar
- **Load balancing** - eBPF-based load balancing with less latency than iptables
- **Hubble** - Layer 7 visibility into Dapr HTTP/gRPC traffic

## Installing Cilium with Hubble

```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all \
  "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz"
tar xzvf cilium-linux-amd64.tar.gz -C /usr/local/bin

# Install Cilium in the cluster
cilium install --version 1.15.0 \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

## Enforcing Dapr Sidecar-Only Communication

Use a Cilium NetworkPolicy to ensure all traffic flows through the Dapr sidecar:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: dapr-sidecar-only
spec:
  endpointSelector:
    matchLabels:
      dapr.io/app-id: "order-processor"
  ingress:
    - fromEndpoints:
        - matchLabels:
            dapr.io/app-id: "api-gateway"  # Only allow from specific Dapr apps
      toPorts:
        - ports:
            - port: "3500"    # Dapr HTTP port only
              protocol: TCP
```

This prevents direct pod-to-pod communication that bypasses Dapr's mTLS and tracing.

## Observing Dapr Traffic with Hubble

Enable Hubble relay and use the CLI:

```bash
# Port-forward Hubble relay
cilium hubble port-forward &

# Observe Dapr sidecar-to-sidecar communication
hubble observe \
  --type l7 \
  --label "dapr.io/enabled=true" \
  --follow

# Filter by namespace
hubble observe \
  --namespace production \
  --protocol http \
  --follow
```

Output shows each HTTP request between Dapr sidecars with latency, response code, and direction.

## Reducing Sidecar Overhead with eBPF Acceleration

Configure Cilium's kube-proxy replacement to accelerate Dapr sidecar-to-sidecar traffic:

```yaml
# cilium-config values
kubeProxyReplacement: strict
loadBalancer:
  algorithm: maglev
  mode: dsr        # Direct Server Return for reduced latency
bpf:
  masquerade: true
  hostLegacyRouting: false
```

This replaces iptables with eBPF programs, reducing latency on the network path by 10-30%.

## Monitoring with Network Flow Metrics

Collect eBPF-level network metrics alongside Dapr metrics:

```bash
# Hubble metrics (add to Prometheus scrape config)
# Default Hubble metrics port: 9965

# Drop rate due to network policy violations
hubble_drop_total

# HTTP request duration at the network level
hubble_http_request_duration_seconds
```

Correlate Hubble HTTP metrics with Dapr's `dapr_http_client_roundtrip_latency_ms` to identify whether latency is in the Dapr sidecar or in the network path.

## Summary

Combining Dapr with eBPF networking via Cilium provides defense in depth: Dapr handles application-level concerns while Cilium enforces network policies at the kernel level, preventing traffic from bypassing the Dapr sidecar. Hubble adds Layer 7 visibility into Dapr HTTP and gRPC traffic, and eBPF-based kube-proxy replacement reduces network latency on the critical path between sidecars.
