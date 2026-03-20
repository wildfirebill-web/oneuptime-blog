# How to Troubleshoot DNS Resolution in Kubernetes Pods

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, Kubernetes, CoreDNS, Troubleshooting, Linux, Networking

Description: Diagnose and fix DNS resolution failures in Kubernetes pods including CoreDNS issues, service discovery failures, and search domain configuration problems.

## Introduction

DNS in Kubernetes is managed by CoreDNS, which resolves both Kubernetes service names (like `my-service.my-namespace.svc.cluster.local`) and external domains. DNS failures in pods manifest as connection errors, slow service startup, or intermittent timeouts. Kubernetes DNS issues have several unique causes not found in regular Linux environments.

## Verify DNS is Working in a Pod

```bash
# Run a temporary debugging pod with DNS tools:
kubectl run dns-debug --image=infoblox/dnstools:latest --rm -it --restart=Never -- bash

# Inside the pod:
# Test service resolution:
nslookup kubernetes.default.svc.cluster.local
dig kubernetes.default.svc.cluster.local

# Test external resolution:
nslookup google.com
dig google.com

# Check pod's DNS configuration:
cat /etc/resolv.conf
# Should contain:
# nameserver 10.96.0.10   (CoreDNS service ClusterIP)
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

## CoreDNS Health Check

```bash
# Check CoreDNS pod status:
kubectl -n kube-system get pods -l k8s-app=kube-dns
# All pods should be Running

# Check CoreDNS logs:
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50

# Check CoreDNS service:
kubectl -n kube-system get svc kube-dns
# CLUSTER-IP should match what's in /etc/resolv.conf of pods

# Test CoreDNS directly:
COREDNS_IP=$(kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.clusterIP}')
kubectl run test-dns --image=busybox --rm -it --restart=Never -- \
  nslookup kubernetes.default $COREDNS_IP
```

## Common DNS Failures

```bash
# Failure 1: NXDOMAIN for service names
# Cause: wrong search domain or ndots setting
# Debug:
kubectl run debug --image=busybox --rm -it --restart=Never -- sh
# Inside:
nslookup my-service.my-namespace    # Short form
nslookup my-service.my-namespace.svc.cluster.local.  # FQDN with trailing dot

# Failure 2: DNS timeout (CoreDNS unreachable)
# Check: can the pod reach CoreDNS?
kubectl run debug --image=busybox --rm -it --restart=Never -- sh
# nc -zv 10.96.0.10 53   # Test TCP
# nc -zuv 10.96.0.10 53  # Test UDP

# Failure 3: Slow DNS causing 5-second delays
# This is caused by IPv4/IPv6 race conditions with 5-second timeout
# Symptom: getaddrinfo() takes exactly 5 seconds
# Fix in CoreDNS Corefile: add forward plugin with prefer-udp
```

## CoreDNS Configuration

```bash
# View current CoreDNS configuration:
kubectl -n kube-system get configmap coredns -o yaml

# Example Corefile:
# .:53 {
#     errors
#     health
#     ready
#     kubernetes cluster.local in-addr.arpa ip6.arpa {
#         pods insecure
#         fallthrough in-addr.arpa ip6.arpa
#     }
#     prometheus :9153
#     forward . /etc/resolv.conf {
#         max_concurrent 1000
#     }
#     cache 30
#     loop
#     reload
#     loadbalance
# }

# Edit CoreDNS config:
kubectl -n kube-system edit configmap coredns

# Restart CoreDNS to apply changes:
kubectl -n kube-system rollout restart deployment coredns
```

## Fix: ndots and Search Domain Optimization

```yaml
# Pods with ndots:5 try 5 search domains before falling back to absolute lookup
# This causes 5 extra DNS queries for external domains!

# For pods that resolve mostly external domains, reduce ndots:
# In pod spec:
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "1"   # Try as absolute domain if it has 1+ dots
    searches:
      - default.svc.cluster.local    # Keep only what you need
      - svc.cluster.local
  dnsPolicy: ClusterFirst
```

## Fix: DNS Cache in Applications

```bash
# Kubernetes service IPs don't change, but external DNS changes
# Many apps (Java, Go) cache DNS results forever or for a long time
# Kubernetes DNS uses 30-second TTLs (CoreDNS cache setting)

# For Java applications:
# Add: -Dnetworkaddress.cache.ttl=30   (respect DNS TTL)
# Or: -Dnetworkaddress.cache.ttl=10   (aggressive re-resolve)

# Check application DNS behavior:
strace -p $POD_PID -e trace=network 2>&1 | grep "connect\|getsockname" | head -20
```

## Conclusion

Kubernetes DNS troubleshooting starts with checking CoreDNS pod health and logs, then testing resolution from inside the affected pod using a debug container. For service discovery failures, verify the full FQDN (`service.namespace.svc.cluster.local`) works before debugging the short form. The most common performance issue is `ndots:5` causing multiple lookup attempts for external domains — reduce to `ndots:1` for pods that primarily resolve external names. Monitor CoreDNS metrics at `:9153` for cache hit rates and error rates.
