# How to Configure K3s with CoreDNS Custom Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, CoreDNS, DNS, Networking, DevOps

Description: Learn how to customize CoreDNS configuration in K3s to support custom DNS zones, forwarding rules, search domains, and external DNS resolution.

## Introduction

K3s deploys CoreDNS as the cluster DNS provider. While the default configuration handles most use cases, custom DNS requirements — such as resolving internal company domains, overriding specific hostnames, or forwarding DNS queries to custom resolvers — require modifying the CoreDNS ConfigMap. This guide covers common CoreDNS customization scenarios in K3s.

## Understanding CoreDNS Configuration

CoreDNS configuration (Corefile) uses a plugin-based architecture where plugins are chained to handle DNS queries. K3s manages CoreDNS through a HelmChart, so customizations must be done carefully to persist across upgrades.

## View Current CoreDNS Configuration

```bash
# View the current CoreDNS ConfigMap
kubectl get configmap coredns -n kube-system -o yaml

# The Corefile is in the 'data.Corefile' key
kubectl get configmap coredns -n kube-system \
  -o jsonpath='{.data.Corefile}'
```

## Step 1: Add Custom DNS Records (Hosts Plugin)

Override specific hostnames with custom IPs:

```bash
# Edit the CoreDNS ConfigMap
kubectl edit configmap coredns -n kube-system
```

Add a custom hosts block:

```yaml
# CoreDNS Corefile with custom hosts
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready

        # Custom hosts overrides
        hosts /etc/coredns/custom-hosts.txt {
            10.0.0.100  internal.example.com
            10.0.0.101  api.internal.example.com
            fallthrough
        }

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

## Step 2: Forward Internal Domains to Custom DNS Servers

Route queries for specific domains to your corporate DNS:

```yaml
# coredns-custom-config.yaml
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
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }

    # Forward corp.example.com queries to internal corporate DNS
    corp.example.com:53 {
        errors
        cache 30
        forward . 10.0.0.1 10.0.0.2 {
            prefer_udp
        }
    }

    # Forward internal.local to local DNS server
    internal.local:53 {
        errors
        cache 30
        forward . 192.168.1.1
    }
```

Apply the ConfigMap:

```bash
kubectl apply -f coredns-custom-config.yaml

# Restart CoreDNS to pick up changes
kubectl rollout restart deployment/coredns -n kube-system

# Watch pods restart
kubectl rollout status deployment/coredns -n kube-system
```

## Step 3: Add Custom Search Domains

Configure additional search domains for pods:

```yaml
# This is typically done via the kubelet config or pod spec
# In a pod spec, you can override DNS config:
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    # Additional search domains
    searches:
      - corp.example.com
      - internal.example.com
    # Custom DNS options
    options:
      - name: ndots
        value: "5"
      - name: timeout
        value: "2"
      - name: attempts
        value: "3"
  containers:
    - name: app
      image: nginx:latest
```

## Step 4: Use ConfigMap for Persistent Custom Configuration

K3s may overwrite the CoreDNS ConfigMap during upgrades. Use a separate ConfigMap for custom configuration:

```yaml
# coredns-custom.yaml
---
# Custom configuration ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  # Custom hosts file
  custom-hosts.txt: |
    10.0.0.100 internal.example.com
    10.0.0.101 api.internal.example.com
    10.0.0.102 db.internal.example.com

  # Custom Corefile additions
  custom.server: |
    corp.example.com:53 {
        errors
        cache 30
        forward . 10.0.0.1 {
            prefer_udp
        }
    }
```

Mount the custom ConfigMap in CoreDNS:

```bash
# Patch the CoreDNS deployment to mount custom config
kubectl patch deployment coredns -n kube-system --patch '
{
  "spec": {
    "template": {
      "spec": {
        "volumes": [
          {
            "name": "coredns-custom",
            "configMap": {
              "name": "coredns-custom"
            }
          }
        ],
        "containers": [
          {
            "name": "coredns",
            "volumeMounts": [
              {
                "name": "coredns-custom",
                "mountPath": "/etc/coredns/custom"
              }
            ]
          }
        ]
      }
    }
  }
}'
```

## Step 5: Add the Import Plugin for Modular Config

Use CoreDNS's `import` plugin to include additional config files:

```yaml
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
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
        # Import all .server files from the custom directory
        import /etc/coredns/custom/*.server
    }
```

## Step 6: Use HelmChartConfig to Persist Changes

For K3s-managed CoreDNS, use HelmChartConfig:

```yaml
# /var/lib/rancher/k3s/server/manifests/coredns-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: coredns
  namespace: kube-system
spec:
  valuesContent: |-
    servers:
    - zones:
      - zone: .
      port: 53
      plugins:
      - name: errors
      - name: health
        configBlock: |-
          lameduck 5s
      - name: ready
      - name: kubernetes
        parameters: cluster.local in-addr.arpa ip6.arpa
        configBlock: |-
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
          ttl 30
      - name: prometheus
        parameters: 0.0.0.0:9153
      - name: forward
        parameters: . 10.0.0.1 10.0.0.2
        configBlock: |-
          max_concurrent 1000
      - name: cache
        parameters: 30
      - name: loop
      - name: reload
      - name: loadbalance
```

## Step 7: Test DNS Resolution

```bash
# Deploy a test pod
kubectl run dns-test --image=busybox:1.35 --restart=Never -- sleep 3600

# Test Kubernetes service DNS resolution
kubectl exec dns-test -- nslookup kubernetes.default.svc.cluster.local

# Test custom domain resolution
kubectl exec dns-test -- nslookup internal.example.com

# Test external DNS resolution
kubectl exec dns-test -- nslookup google.com

# Test corporate domain forwarding
kubectl exec dns-test -- nslookup service.corp.example.com

# Check DNS configuration in the pod
kubectl exec dns-test -- cat /etc/resolv.conf

# Clean up
kubectl delete pod dns-test
```

## Conclusion

CoreDNS in K3s is highly configurable through its plugin system. Common customizations include adding custom host entries, forwarding internal domains to corporate DNS servers, and adjusting search domains. For K3s clusters, using `HelmChartConfig` is the recommended approach for persistent customizations that survive Traefik/CoreDNS upgrades. Always test DNS changes with a test pod before deploying to production to ensure resolution works as expected.
