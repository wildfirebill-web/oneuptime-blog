# How to Set Up Network Address Translation in Rancher (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, NAT, Network, Kubernetes, Egress, SNAT

Description: Configure source and destination NAT in Rancher for pod internet egress, external service access, and managing IP address translation in Kubernetes clusters.

## Introduction

NAT in Kubernetes serves multiple purposes: SNAT for pods to communicate with external services, Egress NAT to present a predictable outbound IP, and DNAT for incoming load balancer traffic. Understanding and configuring NAT correctly is essential for production deployments.

## Types of NAT in Kubernetes

1. **SNAT** (Source NAT): Pods use node IP for outbound connections
2. **Egress NAT**: Specific pods use a dedicated egress IP
3. **DNAT** (Destination NAT): Kubernetes Services perform DNAT for load balancing

## Step 1: Configure SNAT for Pod Internet Access

By default, pods use SNAT via the node's primary IP for internet access. Verify this is working:

```bash
# From a pod, check external connectivity

kubectl exec -it test-pod -- curl -s ifconfig.me
# Should return the node's IP address (SNAT)
```

## Step 2: Configure Dedicated Egress IP with Calico

```yaml
# egress-ip.yaml - Assign a specific IP for all pods in a namespace
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: egress-pool
spec:
  cidr: 203.0.113.0/29    # Public IPs allocated for egress
  ipipMode: Never
  vxlanMode: Never
  natOutgoing: false       # Don't SNAT (use the actual pool IP)
  disabled: false
---
apiVersion: projectcalico.org/v3
kind: EgressIPSet
metadata:
  name: payments-egress
spec:
  cidr: 203.0.113.1/32    # Specific IP for payments namespace
```

## Step 3: Configure Egress Gateway

For fine-grained egress control, use Calico's Egress Gateway:

```yaml
# egress-gateway.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: egress-gateway
  namespace: production
spec:
  replicas: 2
  template:
    metadata:
      annotations:
        cni.projectcalico.org/ipAddrs: '["203.0.113.1"]'
    spec:
      containers:
        - name: egress
          image: calico/egress-gateway:latest
```

## Step 4: Disable SNAT for Specific Pods

```yaml
# Disable SNAT for pods needing to present their original IP
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: no-nat-pool
spec:
  cidr: 10.244.10.0/24
  natOutgoing: false    # Disable SNAT for this pool
  nodeSelector: all()
```

## Step 5: Configure DNAT Rules for Custom Services

```bash
# Add custom DNAT rules via iptables (applied on all nodes)
# Forward port 8080 to a specific pod IP
iptables -t nat -A PREROUTING \
  -p tcp --dport 8080 \
  -j DNAT --to-destination 10.244.1.5:80

# Make persistent via DaemonSet or init container
```

## Step 6: Monitor NAT Table

```bash
# View current NAT rules (on a node)
iptables -t nat -L -n -v | grep -E "SNAT|DNAT|MASQUERADE"

# Count NAT conntrack entries
cat /proc/sys/net/netfilter/nf_conntrack_count
```

## Conclusion

NAT management in Rancher primarily involves controlling which IP address pods present for outbound connections. Calico's Egress IP and Egress Gateway features provide the most flexible control for regulated environments that require whitelisting specific source IPs at external firewalls.
