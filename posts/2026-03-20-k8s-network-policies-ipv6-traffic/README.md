# How to Configure Kubernetes Network Policies for IPv6 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, IPv6, NetworkPolicy, Security, Calico, Cilium

Description: A practical guide to writing Kubernetes NetworkPolicy resources that correctly control IPv6 pod traffic in dual-stack and IPv6-only clusters.

Kubernetes NetworkPolicy resources enforce pod-level firewall rules. In dual-stack clusters, policies must explicitly account for both IPv4 and IPv6 traffic, or use a CNI plugin (like Calico or Cilium) that enforces policies across both address families automatically.

## How NetworkPolicy Works with IPv6

Standard Kubernetes NetworkPolicy uses `ipBlock` selectors which support both IPv4 and IPv6 CIDR notation. When your CNI plugin enforces these policies, it programs both `iptables` and `ip6tables` (or eBPF programs) accordingly.

## Step 1: Default Deny All IPv6 Ingress

Apply a default-deny policy to restrict all incoming IPv6 traffic to pods in a namespace:

```yaml
# default-deny-ingress.yaml - Deny all ingress traffic (covers both IPv4 and IPv6)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}       # Applies to ALL pods in the namespace
  policyTypes:
    - Ingress
  # No ingress rules = deny all ingress
```

## Step 2: Allow Specific IPv6 CIDR Blocks

Allow traffic only from a specific IPv6 subnet (e.g., your internal monitoring network):

```yaml
# allow-monitoring-ipv6.yaml - Allow ingress from a specific IPv6 CIDR
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-ipv6
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            # Allow from the internal monitoring IPv6 subnet
            cidr: "fd00:monitoring::/64"
      ports:
        - protocol: TCP
          port: 9090
```

## Step 3: Allow Pod-to-Pod IPv6 Traffic Within a Namespace

```yaml
# allow-same-namespace.yaml - Allow all traffic between pods in the same namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}   # Any pod in the same namespace
```

## Step 4: Egress Policy Blocking IPv6 External Traffic

```yaml
# restrict-egress-ipv6.yaml - Restrict IPv6 egress to internal cluster only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-ipv6-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: restricted-app
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            # Allow egress to the internal pod CIDR
            cidr: "fd00:10:244::/56"
        - ipBlock:
            # Allow egress to the internal service CIDR
            cidr: "fd00:10:96::/112"
    - ports:
        # Always allow DNS
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

## Step 5: Dual-Stack Policy (IPv4 + IPv6)

For a dual-stack cluster, specify both CIDRs to cover both address families:

```yaml
# allow-dual-stack-lb.yaml - Allow traffic from dual-stack load balancer range
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-lb-access
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: "10.0.0.0/8"      # IPv4 load balancer range
        - ipBlock:
            cidr: "fd00:lb::/64"    # IPv6 load balancer range
      ports:
        - protocol: TCP
          port: 80
        - protocol: TCP
          port: 443
```

## Verifying Policy Enforcement

```bash
# Test that traffic is blocked from an unauthorized pod
kubectl run test-blocked --image=busybox:1.36 --restart=Never \
  -- wget --timeout=5 -O- http://[<target-ipv6>]:80/

# The connection should time out or be refused
kubectl logs test-blocked
```

## CNI Plugin Considerations

- **Calico**: Fully supports IPv6 NetworkPolicy, including GlobalNetworkPolicy for cluster-wide rules
- **Cilium**: Supports IPv6 NetworkPolicy and offers richer L7 policies via CiliumNetworkPolicy
- **Flannel**: Does **not** support NetworkPolicy — you need a separate policy engine

Writing explicit NetworkPolicy objects for both IPv4 and IPv6 CIDRs ensures your security posture is complete in dual-stack Kubernetes clusters.
