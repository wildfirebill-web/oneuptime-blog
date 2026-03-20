# How to Configure Cilium IPv6 Service Load Balancing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cilium, IPv6, Load Balancing, Kubernetes, Kube-proxy, eBPF

Description: Configure Cilium's eBPF-based IPv6 service load balancing to replace kube-proxy, enable DSR, and configure session affinity for IPv6 services.

## Introduction

Cilium replaces kube-proxy for service load balancing using eBPF programs that operate at the socket level and XDP layer. For IPv6, this provides efficient ClusterIP, NodePort, and LoadBalancer service handling without iptables overhead.

## Enable kube-proxy Replacement

```bash
# Install Cilium with kube-proxy replacement

helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=api.cluster.example.com \
  --set k8sServicePort=6443 \
  --set ipv6.enabled=true \
  --set ipv4.enabled=true

# Verify kube-proxy replacement is active
cilium status | grep "KubeProxy"
# KubeProxyReplacement:    True   [eth0 (Direct Routing)]
```

## Dual-Stack Service with IPv6 ClusterIP

```yaml
# Kubernetes Service with dual-stack
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ipFamilyPolicy: RequireDualStack
  ipFamilies:
    - IPv6
    - IPv4
  selector:
    app: my-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
  type: ClusterIP
```

```bash
# Verify dual-stack ClusterIPs assigned
kubectl get svc my-service -o jsonpath='{.spec.clusterIPs}'
# ["10.96.1.100", "fd00:10:96::100"]

# Test IPv6 ClusterIP from a pod
kubectl exec -it test-pod -- curl -6 http://[fd00:10:96::100]/health

# Inspect Cilium eBPF load balancer entries for this service
cilium service list | grep "fd00:10:96::100"
```

## NodePort with IPv6

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  ipFamilyPolicy: SingleStack
  ipFamilies:
    - IPv6
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

```bash
# Access via any node's IPv6 address on the nodePort
NODE_IPV6=$(kubectl get node node1 -o jsonpath='{.status.addresses[?(@.type=="ExternalIP")].address}')
curl -6 "http://[$NODE_IPV6]:30080/"

# Cilium handles NodePort via eBPF:
cilium bpf nodeport list | grep "30080"
```

## Direct Server Return (DSR) for NodePort

```bash
# DSR: backend pods respond directly to clients (skips SNAT)
# Requires BGP or L2 announcement

helm upgrade cilium cilium/cilium \
  --reuse-values \
  --set loadBalancer.mode=dsr \
  --set ipv4.enabled=true \
  --set ipv6.enabled=true

# DSR works best with:
# - External LoadBalancer routing traffic directly to nodes
# - BGP-based LoadBalancer (metallb or cilium's BGP)
```

## Session Affinity (Sticky Sessions)

```yaml
# Configure session affinity per service
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: stateful-app
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600  # 1 hour
  ipFamilies:
    - IPv6
  ports:
    - port: 443
      targetPort: 8443
```

```bash
# Verify session affinity in eBPF
cilium service list | grep "sticky-service"
# Shows: affinity timeout = 3600s

# Cilium stores per-client affinity in eBPF maps
cilium bpf lb maglev list | grep "sticky"
```

## Monitoring Load Balancer Health

```bash
# Check backend health for an IPv6 service
cilium service get <service-id>

# Monitor active connections per backend
cilium bpf lb list --backends

# Hubble: observe load balancer decisions
hubble observe --type l4 --verdict FORWARDED \
  --to-port 80 --ip-version ipv6 | head -20

# Prometheus metrics
# cilium_services_events_total{action="add"} - services created
# cilium_backend_state_count{state="active"} - healthy backends
```

## Conclusion

Cilium's eBPF load balancer replaces kube-proxy for IPv6 services with lower latency and higher throughput. Enable kube-proxy replacement, configure dual-stack services, and optionally use DSR for optimal return path routing. Session affinity uses eBPF maps for fast per-client routing. Monitor backend availability and load distribution with OneUptime and Hubble.
