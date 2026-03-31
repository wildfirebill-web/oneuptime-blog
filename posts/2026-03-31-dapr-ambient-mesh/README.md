# How to Use Dapr with Ambient Mesh

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Ambient Mesh, Istio, Service Mesh, Kubernetes, Networking

Description: Learn how to run Dapr alongside Istio Ambient Mesh to get mTLS and L4 observability without per-pod sidecars, reducing resource overhead in your cluster.

---

## What Is Ambient Mesh?

Istio Ambient Mesh is a sidecar-free service mesh architecture. Instead of injecting an Envoy proxy into every pod, Ambient uses:
- **ztunnel** - A per-node DaemonSet for L4 mTLS and observability
- **waypoint proxy** - An optional per-namespace proxy for L7 policies

This reduces the per-pod overhead of Istio while maintaining security guarantees.

## Why Combine Dapr with Ambient Mesh

Dapr provides application-level APIs (state, pub/sub, secrets, workflow). Istio Ambient provides network-level mTLS and observability. Together they form a complete platform without the duplication of two sets of per-pod sidecars.

| Feature | Dapr | Ambient Mesh |
|---------|------|--------------|
| Service invocation | Yes | Complement |
| mTLS | Yes (Dapr mTLS) | Yes (ztunnel mTLS) |
| L7 traffic policies | Limited | Via waypoint |
| Pub/sub | Yes | No |
| State management | Yes | No |
| Network observability | Limited | Full L4/L7 |

## Installing Istio Ambient Mode

```bash
# Download Istio with ambient profile
curl -L https://istio.io/downloadIstio | sh -
cd istio-*/

# Install with ambient profile
istioctl install --set profile=ambient --confirm

# Verify ztunnel is running
kubectl get pods -n istio-system -l app=ztunnel
```

## Enabling Ambient for Your Namespace

Label the namespace to opt into ambient mesh:

```bash
kubectl label namespace production \
  istio.io/dataplane-mode=ambient
```

Pods in this namespace now have ztunnel handling L4 mTLS without Envoy sidecars.

## Running Dapr Alongside Ambient Mesh

Dapr sidecars coexist with ambient mesh. Both can be enabled simultaneously:

```yaml
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "8080"
        # Dapr mTLS handles app-to-dapr, ztunnel handles pod-to-pod
```

Dapr's own mTLS (via the Dapr Sentry service) handles sidecar-to-sidecar encryption. Ambient ztunnel provides additional L4 encryption for any traffic not going through Dapr.

## Disabling Dapr mTLS When Using Ambient mTLS

If you want to rely solely on Ambient for mTLS (simplifying cert management), disable Dapr mTLS:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  mtls:
    enabled: false
```

Only do this if Ambient mTLS is confirmed working and covers all network paths.

## Observing Traffic with Ambient + Dapr

Use `istioctl` to observe L4 flows between Dapr sidecars:

```bash
# Watch L4 metrics for Dapr sidecar port
kubectl exec -n istio-system dapr-ztunnel-xxx -- \
  curl http://localhost:15020/metrics | grep dapr

# Check waypoint proxy access logs for L7 visibility
kubectl logs -n production -l app=waypoint --follow
```

## Resource Comparison

```yaml
resource_savings_with_ambient:
  per_pod:
    envoy_sidecar_removed: true
    memory_saved: "~80 MB per pod"
    cpu_saved: "~100m millicores per pod"
  cluster_wide:
    ztunnel_added: "~50 MB per node (DaemonSet)"
  net_savings_for_100_pods:
    memory: "~7.5 GB"
    note: "Dapr sidecar still present at ~35 MB per pod"
```

## Summary

Running Dapr with Istio Ambient Mesh eliminates the need for per-pod Envoy proxies while retaining L4 mTLS from ztunnel. Dapr sidecars continue to provide application-level building blocks (state, pub/sub, secrets), while ambient handles network-level security and observability. This combination reduces per-pod memory overhead by approximately 80 MB compared to Dapr plus traditional Istio sidecar mode.
