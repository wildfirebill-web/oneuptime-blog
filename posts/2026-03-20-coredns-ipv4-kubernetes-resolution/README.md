# How to Configure CoreDNS for IPv4 Name Resolution in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: CoreDNS, Kubernetes, IPv4, DNS, Service Discovery, Networking

Description: Configure CoreDNS for IPv4 DNS name resolution in Kubernetes, including custom forwarding rules, caching, and stub zones for internal domain resolution.

CoreDNS is the default DNS server in Kubernetes. It resolves service names to ClusterIPs and provides configurable forwarding to external DNS servers.

## Viewing the CoreDNS Configuration

```bash
# CoreDNS configuration is stored in a ConfigMap

kubectl get configmap coredns -n kube-system -o yaml

# Default Corefile:
# .:53 {
#     errors
#     health { lameduck 5s }
#     ready
#     kubernetes cluster.local in-addr.arpa ip6.arpa {
#        pods insecure
#        fallthrough in-addr.arpa ip6.arpa
#        ttl 30
#     }
#     prometheus :9153
#     forward . /etc/resolv.conf {
#        max_concurrent 1000
#     }
#     cache 30
#     loop
#     reload
#     loadbalance
# }
```

## How Kubernetes DNS Resolution Works

```text
Pod queries: my-service.my-namespace.svc.cluster.local
CoreDNS searches kubernetes zone: cluster.local
Returns: ClusterIP 10.96.45.123
Pod connects to 10.96.45.123
```

Short names work because pods have a search domain list: `my-namespace.svc.cluster.local svc.cluster.local cluster.local`

## Adding Stub Zones for Internal Domains

Forward queries for internal corporate domains to your internal DNS server:

```yaml
# Edit the CoreDNS ConfigMap
kubectl edit configmap coredns -n kube-system
```

```text
# Corefile with stub zone for internal domain
.:53 {
    errors
    health { lameduck 5s }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    # Forward queries for corp.example.com to internal DNS
    forward corp.example.com 192.168.1.53 {
       prefer_udp
    }
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    prometheus :9153
    cache 30
    loop
    reload
    loadbalance
}
```

```bash
# Reload CoreDNS after ConfigMap update
kubectl rollout restart deployment/coredns -n kube-system
```

## Increasing DNS Cache TTL

```text
.:53 {
    # Increase cache TTL from 30s to 300s for better performance
    cache 300
    ...
}
```

## Configuring Negative Caching

```text
.:53 {
    # Cache NXDOMAIN responses for 30 seconds (reduces failed lookups)
    cache 300 {
        success 9984 300
        denial 9984 30
    }
    ...
}
```

## Enabling NodeLocal DNSCache

NodeLocal DNSCache runs a DNS cache on each node to reduce CoreDNS load:

```bash
# Download and apply NodeLocal DNSCache manifest
wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml

# Replace the IP placeholders with your cluster DNS IPs
KUBEDNS=$(kubectl get svc kube-dns -n kube-system -o jsonpath='{.spec.clusterIP}')
DOMAIN=cluster.local
LOCALDNS=169.254.20.10

sed -i "s/__PILLAR__LOCAL__DNS__/$LOCALDNS/g" nodelocaldns.yaml
sed -i "s/__PILLAR__DNS__DOMAIN__/$DOMAIN/g" nodelocaldns.yaml
sed -i "s/__PILLAR__DNS__SERVER__/$KUBEDNS/g" nodelocaldns.yaml

kubectl apply -f nodelocaldns.yaml
```

## Verifying CoreDNS Operation

```bash
# Check CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Test DNS resolution from inside the cluster
kubectl run dnstest --image=alpine --restart=Never -- sleep 3600
kubectl exec dnstest -- nslookup kubernetes.default.svc.cluster.local

# Check CoreDNS logs for query errors
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

Well-tuned CoreDNS is critical for cluster performance - DNS is the first step in every service discovery operation.
