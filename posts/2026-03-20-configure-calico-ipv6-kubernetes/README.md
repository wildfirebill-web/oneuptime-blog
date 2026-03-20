# How to Configure Calico for IPv6 in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Calico, IPv6, Kubernetes, CNI, BGP, NetworkPolicy, Dual-Stack

Description: Configure Calico CNI for IPv6 and dual-stack Kubernetes clusters, including BGP-based routing, IPPool configuration, and NetworkPolicy for IPv6.

## Introduction

Calico is a popular CNI for Kubernetes that supports BGP-based pod routing. It has strong IPv6 support including IPv6 IPPools, BGP peering, and dual-stack networking. Calico's network policies work natively with IPv6 addresses.

## Step 1: Install Calico with IPv6 Support

```bash
# Install the Calico operator

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# Configure dual-stack installation
kubectl create -f - << 'EOF'
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - blockSize: 26
        cidr: 10.244.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
      - blockSize: 122
        cidr: fd00:10:244::/48
        encapsulation: None
        natOutgoing: Disabled
        nodeSelector: all()
  # Dual-stack
  serviceCIDRs:
    - 10.96.0.0/12
    - fd00:10:96::/108
EOF
```

## Step 2: Configure IPv6 IPPool

```yaml
# calico-ipv6-pool.yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv6-pool
spec:
  cidr: fd00:10:244::/48
  blockSize: 122       # /122 = 4 addresses per block
  ipipMode: Never      # No IPIP for IPv6
  vxlanMode: Never     # No VXLAN for IPv6
  natOutgoing: false   # No NAT for IPv6 (has routable addresses)
  nodeSelector: all()
  disabled: false
```

```bash
# Apply the pool
calicoctl apply -f calico-ipv6-pool.yaml

# Verify pools
calicoctl get ippools

# Check pod IPv6 allocation
calicoctl get ipam blocks --inet6
```

## Step 3: BGP Configuration for IPv6

```yaml
# calico-bgp-config.yaml
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true
  asNumber: 65000
  # Advertise IPv6 pod CIDRs
  serviceClusterIPs:
    - cidr: fd00:10:96::/108  # IPv6 service CIDR

---
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: ipv6-upstream-peer
spec:
  peerIP: 2001:db8::upstream
  asNumber: 65001
  # Peer over IPv6
  keepOriginalNextHop: false
```

## Step 4: NetworkPolicy for IPv6

```yaml
# netpolicy-ipv6.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ipv6-external
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow from IPv6 clients
    - from:
        - ipBlock:
            cidr: 2001:db8:client::/48
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # Allow to IPv6 DNS
    - to:
        - ipBlock:
            cidr: 2001:db8::53/128
      ports:
        - protocol: UDP
          port: 53
```

## Step 5: Calico GlobalNetworkPolicy for IPv6

```yaml
# calico-global-policy.yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-ipv6-link-local
spec:
  selector: all()
  order: 100
  ingress:
    # Block link-local source addresses from outside (anti-spoofing)
    - action: Deny
      source:
        nets:
          - "fe80::/10"
      destination: {}
  egress:
    - action: Allow
```

## Step 6: Verify and Test

```bash
# Check Calico status
calicoctl node status

# Verify IPv6 addresses on pods
kubectl get pods -o wide
# Each pod should have an IPv4 and IPv6 address in dual-stack

# Test IPv6 pod-to-pod
kubectl exec -it pod-a -- ping6 fd00:10:244::5

# Test IPv6 service access
kubectl exec -it pod-a -- curl -6 http://[fd00:10:96::10]:8080/

# View BGP routes including IPv6
calicoctl get bgp routes
```

## Conclusion

Calico supports IPv6 via dedicated IPPools with IPv6 CIDRs and BGP advertisement of IPv6 pod routes. Configure `blockSize: 122` for IPv6 blocks (/122 = 4 usable addresses per node block). NetworkPolicy works with IPv6 `ipBlock` CIDRs. Monitor Calico agent health, BGP session state, and IPv6 IPAM utilization with OneUptime's infrastructure checks.
