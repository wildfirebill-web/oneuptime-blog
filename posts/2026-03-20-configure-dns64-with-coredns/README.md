# How to Configure DNS64 with CoreDNS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS64, CoreDNS, Kubernetes, NAT64

Description: Learn how to configure the DNS64 plugin in CoreDNS to synthesize AAAA records for IPv4-only domains, with examples for both standalone and Kubernetes deployments.

## Why CoreDNS for DNS64?

CoreDNS is the default DNS server in Kubernetes and a popular choice for cloud-native deployments. Its DNS64 plugin makes it straightforward to enable NAT64+DNS64 in Kubernetes clusters with IPv6-only pods or in edge deployments.

## Installing CoreDNS

CoreDNS ships with the DNS64 plugin built in since version 1.7.0. No additional installation is needed:

```bash
# Check CoreDNS version

coredns --version

# Download CoreDNS binary if not installed
# Visit https://github.com/coredns/coredns/releases
wget https://github.com/coredns/coredns/releases/download/v1.11.1/coredns_1.11.1_linux_amd64.tgz
tar -xzf coredns_1.11.1_linux_amd64.tgz
mv coredns /usr/local/bin/
```

## Basic Corefile Configuration for DNS64

CoreDNS is configured via a `Corefile`. Add the `dns64` plugin to a server block:

```corefile
# /etc/coredns/Corefile

# Handle all DNS queries on port 53
.:53 {
    # Enable DNS64 with the well-known NAT64 prefix
    dns64 {
        # The NAT64 prefix to embed IPv4 addresses into
        prefix 64:ff9b::/96

        # translate-all: also synthesize when AAAA exists (default: false)
        # translate_all false
    }

    # Forward unresolved queries to upstream DNS resolvers
    forward . 8.8.8.8 8.8.4.4 {
        prefer_udp
    }

    # Cache responses for performance
    cache 300

    # Log queries for debugging
    log

    # Return SERVFAIL on errors
    errors
}
```

## DNS64 Plugin Options

The CoreDNS DNS64 plugin supports these directives:

```corefile
dns64 {
    # NAT64 prefix (required) - default is 64:ff9b::/96
    prefix 64:ff9b::/96

    # If true, synthesize AAAA even when native AAAA exists
    # Useful for testing; not recommended for production
    translate_all false
}
```

## Kubernetes ConfigMap Configuration

In Kubernetes, CoreDNS configuration is stored in a ConfigMap in the `kube-system` namespace. To enable DNS64 for in-cluster queries:

```yaml
# Apply this ConfigMap to enable DNS64 in Kubernetes CoreDNS
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
           ttl 30
        }
        # Enable DNS64 for IPv6-only pods connecting to IPv4 services
        dns64 {
            prefix 64:ff9b::/96
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

Apply the updated ConfigMap:

```bash
# Apply the updated ConfigMap
kubectl apply -f coredns-configmap.yaml

# Restart CoreDNS pods to pick up the new configuration
kubectl rollout restart deployment/coredns -n kube-system

# Verify CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

## Testing DNS64 in CoreDNS

```bash
# Test from a pod in an IPv6-only subnet
kubectl run test-dns64 --image=busybox --rm -it -- sh

# Inside the pod: query for a domain with only A records
nslookup example.com
# Should return an address in 64:ff9b::/96

# Using dig from the host against the CoreDNS service
dig AAAA example.com @<coredns-service-ip>

# Expected: 64:ff9b::5db8:d822 (for example.com's 93.184.216.34)
```

## Standalone CoreDNS Deployment

For standalone (non-Kubernetes) deployment, create a systemd service:

```ini
# /etc/systemd/system/coredns.service
[Unit]
Description=CoreDNS DNS Server
After=network.target

[Service]
ExecStart=/usr/local/bin/coredns -conf /etc/coredns/Corefile
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start CoreDNS
systemctl daemon-reload
systemctl enable coredns
systemctl start coredns
systemctl status coredns
```

## Monitoring CoreDNS DNS64

CoreDNS exposes Prometheus metrics on port 9153 by default. Key metrics for DNS64:

```promql
# Total DNS64 synthesis requests
coredns_dns64_requests_total

# Rate of synthesis over 5 minutes
rate(coredns_dns64_requests_total[5m])
```

## Summary

CoreDNS makes DNS64 configuration simple via the `dns64` plugin block in the Corefile. For Kubernetes, update the `coredns` ConfigMap in `kube-system` to add the plugin. CoreDNS is ideal for cloud-native IPv6 deployments where NAT64+DNS64 enables IPv6-only pods to reach IPv4-only external services.
