# How to Use Dapr with IPv6 Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, IPv6, Networking, Kubernetes, Configuration

Description: Configure Dapr to work correctly on IPv6-only and dual-stack Kubernetes clusters, including sidecar binding, component addresses, and DNS resolution.

---

## Dapr and IPv6 Support

As IPv6 adoption grows - particularly in dual-stack Kubernetes environments - it's important to verify that Dapr sidecars bind to the correct interfaces and that component addresses use IPv6-formatted URLs. Dapr has had progressive IPv6 improvements and most scenarios work well with proper configuration.

## IPv6 in Kubernetes Dual-Stack Clusters

First, verify your cluster supports dual-stack:

```bash
# Check cluster network configuration
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDRs}'

# Check pods get IPv6 addresses
kubectl get pods -o wide | head -5
```

## Configuring Dapr for IPv6

By default, the Dapr sidecar binds to `0.0.0.0` (IPv4). On dual-stack clusters, you may need to configure it to bind to `::` for IPv6:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "my-service"
        dapr.io/app-port: "8080"
        dapr.io/config: "appconfig"
    spec:
      containers:
      - name: my-service
        image: my-service:latest
        env:
        - name: DAPR_HOST
          value: "::1"  # IPv6 loopback for sidecar communication
```

## Component Configuration with IPv6 Addresses

When your backing services (Redis, Kafka) use IPv6 addresses:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      # IPv6 addresses in URLs require square brackets
      value: "[fd00::1]:6379"
```

For Kafka pub/sub:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: "[fd00::2]:9092,[fd00::3]:9092"
```

## Testing IPv6 Connectivity

```bash
# Check if Dapr sidecar is listening on IPv6
kubectl exec POD_NAME -c daprd -- ss -6tlnp

# Test Dapr HTTP API over IPv6 loopback
kubectl exec POD_NAME -c app -- \
  curl -g -6 "http://[::1]:3500/v1.0/healthz"

# Test gRPC over IPv6
kubectl exec POD_NAME -c app -- \
  grpcurl -plaintext "[::1]:50001" \
  dapr.proto.runtime.v1.Dapr/GetState \
  -d '{"store_name": "statestore", "key": "test"}'
```

## DNS Resolution in IPv6 Environments

Kubernetes DNS (CoreDNS) supports both A and AAAA records. Ensure CoreDNS is configured for dual-stack:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

## Service Invocation in IPv6 Clusters

Dapr's name resolution component uses Kubernetes DNS for service discovery. On IPv6 clusters, ensure your app's HTTP client supports IPv6:

```javascript
// Node.js - ensure IPv6 support
const http = require('http');
const agent = new http.Agent({ family: 6 });

// Dapr HTTP calls use localhost (::1 in IPv6-only)
const daprUrl = `http://[::1]:3500/v1.0/invoke/target-service/method/endpoint`;
```

## Summary

Running Dapr on IPv6 or dual-stack Kubernetes clusters requires ensuring component addresses use bracket-notation for IPv6 IPs, that CoreDNS is configured to serve AAAA records, and that your application's HTTP clients are IPv6-capable. With these configurations in place, Dapr's sidecar model works transparently across IPv4 and IPv6 network environments.
