# How to Troubleshoot DNS Resolution Issues in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Troubleshooting, DNS, Networking

Description: Diagnose and fix DNS resolution failures in Rancher-managed Kubernetes clusters, including CoreDNS configuration, ndots settings, and upstream resolver problems.

## Introduction

DNS failures in Kubernetes can manifest as pod startup errors, service connectivity failures, or intermittent timeouts. Since Kubernetes relies on CoreDNS for internal service discovery, issues with CoreDNS cascade to virtually every networked workload. This guide covers how to debug and resolve DNS issues in Rancher-managed clusters.

## Step 1: Verify CoreDNS is Running

```bash
# Check CoreDNS pods

kubectl get pods -n kube-system -l k8s-app=kube-dns

# If pods are not Running, describe them for events
kubectl describe pod -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100

# Verify the CoreDNS service and endpoints
kubectl get service -n kube-system kube-dns
kubectl get endpoints -n kube-system kube-dns
```

## Step 2: Test DNS Resolution from a Pod

```bash
# Run a temporary debug pod
kubectl run dns-debug --image=nicolaka/netshoot --restart=Never --rm -it -- bash

# Inside the debug pod:
# Test Kubernetes service DNS
nslookup kubernetes.default.svc.cluster.local
nslookup <your-service>.<namespace>.svc.cluster.local

# Test external DNS
nslookup google.com

# Check the pod's /etc/resolv.conf
cat /etc/resolv.conf
# Should show: nameserver <kube-dns-cluster-ip>
# And: search default.svc.cluster.local svc.cluster.local cluster.local
```

## Step 3: Check CoreDNS Configuration

```bash
# View the CoreDNS ConfigMap
kubectl get configmap -n kube-system coredns -o yaml
```

A typical CoreDNS Corefile looks like:

```text
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

Common issues:

- Missing `forward` stanza (external DNS won't resolve)
- Wrong cluster domain (not `cluster.local`)
- `loop` plugin detecting a forwarding loop

## Step 4: Fix the ndots Setting

High `ndots` values cause excessive DNS lookup attempts before trying the absolute domain name:

```bash
# Check the default ndots setting
kubectl get configmap -n kube-system coredns -o yaml | grep ndots

# In pod spec, you can override ndots:
# (For external domains like "google.com", 5 dots trigger 5 local searches first)
```

```yaml
# Optimize DNS for a specific pod
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"   # Reduce from default 5 to speed up external DNS
      - name: single-request-reopen
        value: ""    # Fix intermittent DNS failures on some kernels
```

## Step 5: Debug with DNS Lookup Tools

```bash
# From inside the cluster, use dig for detailed DNS traces
kubectl run dig-debug --image=tutum/dnsutils --restart=Never --rm -it -- bash

# Detailed lookup with trace
dig @<kube-dns-ip> kubernetes.default.svc.cluster.local +norecurse

# Test upstream resolver connectivity from CoreDNS pod
COREDNS_POD=$(kubectl get pod -n kube-system -l k8s-app=kube-dns -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n kube-system ${COREDNS_POD} -- cat /etc/resolv.conf

# Check if CoreDNS can reach the upstream resolver
kubectl exec -n kube-system ${COREDNS_POD} -- nslookup google.com 8.8.8.8
```

## Step 6: Fix CoreDNS Loop Detection

```bash
# If CoreDNS logs show: "Loop (127.0.0.1:43465 -> :53) detected"
# This happens when the node's /etc/resolv.conf points to 127.0.0.1

# Option 1: Override CoreDNS to use a specific upstream
kubectl edit configmap -n kube-system coredns
# Change: forward . /etc/resolv.conf
# To:     forward . 8.8.8.8 8.8.4.4

# Option 2: Fix the node's /etc/resolv.conf (for systemd-resolved nodes)
# On the node:
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

## Step 7: Scale CoreDNS for High Load

```bash
# Check CoreDNS metrics (if Prometheus is installed)
kubectl port-forward -n kube-system service/kube-dns 9153:9153
curl http://localhost:9153/metrics | grep coredns_dns_requests_total

# Scale CoreDNS replicas
kubectl scale deployment -n kube-system coredns --replicas=3

# Enable NodeLocal DNSCache for improved performance
# (Available as a DaemonSet addon in RKE2/K3s)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
```

## Conclusion

DNS resolution issues in Rancher-managed clusters almost always trace back to CoreDNS misconfiguration, upstream resolver problems, or networking issues between pods and the CoreDNS service. Using a debug pod with `netshoot` or `dnsutils`, combined with CoreDNS log analysis, will quickly identify the root cause. For production clusters, consider scaling CoreDNS replicas and enabling NodeLocal DNSCache to improve reliability and performance.
