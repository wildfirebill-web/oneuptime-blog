# How to Configure MetalLB BGP Mode for IPv4 Address Announcement

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MetalLB, Kubernetes, IPv4, BGP, Networking, Bare Metal

Description: Configure MetalLB BGP mode to announce Kubernetes service IPv4 addresses to your network routers using BGP for scalable, fault-tolerant load balancing.

BGP mode provides true load balancing across all Kubernetes nodes (ECMP) and enables true multi-path routing to services. It requires BGP-capable routers in your network.

## How MetalLB BGP Mode Works

```
MetalLB Speaker on each node → BGP peers with router
Each node announces service IPs via BGP
Router receives routes to service IP via multiple nodes (ECMP)
Traffic distributed across all nodes at the router level
```

## Step 1: Configure the BGP Peer (Router Side)

On your router (example: a Linux router with FRRouting):

```bash
# On the router, configure BGP to accept peers from Kubernetes nodes
# /etc/frr/frr.conf

router bgp 65000
 bgp router-id 192.168.1.1
 neighbor 192.168.1.10 remote-as 65001  ! Kubernetes node 1
 neighbor 192.168.1.11 remote-as 65001  ! Kubernetes node 2
 neighbor 192.168.1.12 remote-as 65001  ! Kubernetes node 3
 !
 address-family ipv4 unicast
  neighbor 192.168.1.10 activate
  neighbor 192.168.1.11 activate
  neighbor 192.168.1.12 activate
 exit-address-family
```

## Step 2: Configure MetalLB BGP Peer

```yaml
# metallb-bgp-peer.yaml
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: router-peer
  namespace: metallb-system
spec:
  # Your router's IPv4 address
  peerAddress: 192.168.1.1
  # Your router's BGP AS number
  peerASN: 65000
  # MetalLB's ASN (all Kubernetes nodes use the same ASN)
  myASN: 65001
  # Optional: BFD for fast failure detection
  # bfdProfile: fast-bfd
```

```bash
kubectl apply -f metallb-bgp-peer.yaml
```

## Step 3: Create IP Pool and BGP Advertisement

```yaml
# metallb-bgp-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: bgp-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.0/24  # Public or routable IP range for services

---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: bgp-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - bgp-pool
  # Optional: set BGP communities on advertisements
  communities:
  - 65000:1
  # Optional: only advertise from specific nodes
  # nodeSelectors:
  # - matchLabels:
  #     node-role: edge
```

```bash
kubectl apply -f metallb-bgp-config.yaml
```

## Step 4: Verify BGP Session Status

```bash
# Check MetalLB speaker logs for BGP session status
kubectl logs -n metallb-system -l component=speaker | grep -i "bgp\|session\|established"

# Expected: "Established BGP session with 192.168.1.1"

# On the router, verify MetalLB routes are received
# (FRR router):
# show ip bgp
# Should show routes for the service IPs learned from Kubernetes nodes
```

## Testing Load Balancing with ECMP

```bash
# Create a service
kubectl expose deployment my-app --type=LoadBalancer --port=80

# Check the external IP
kubectl get svc my-app
# EXTERNAL-IP: 10.0.0.5

# From outside the cluster, verify traffic reaches different nodes
for i in $(seq 1 10); do curl -s http://10.0.0.5 | grep "Hostname"; done
# Should show different pod hostnames, indicating ECMP distribution
```

BGP mode provides better performance and true load balancing compared to L2 mode, but requires router-level BGP configuration.
