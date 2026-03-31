# How to Use Static IPs with Multus for Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Multus, Static IP, Networking

Description: Learn how to configure static IP addresses for Rook-Ceph pods using Multus NetworkAttachmentDefinitions with static IPAM for predictable, stable network addressing.

---

Static IPs for Ceph pods via Multus provide the most predictable and stable networking for Rook deployments. Unlike DHCP or Whereabouts, static IPAM assigns specific pre-defined IPs to each pod. This is particularly valuable for monitor pods, where IP stability is critical for cluster health.

## When to Use Static IPs

Static IPs are appropriate when:
- You need deterministic IP addressing for Ceph monitors
- Your network team requires documented, pre-approved IP assignments
- You want zero dependency on external IPAM services (no DHCP server, no Whereabouts)
- Troubleshooting requires knowing exactly which IP each daemon will have

The trade-off is that you must manage IP assignments manually - each pod that needs a static IP requires its own configuration.

## Static IPAM in NAD Configuration

The static IPAM type allows specifying exact addresses:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rook-mon-a-network
  namespace: rook-ceph
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "rook-mon-a-network",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "static",
        "addresses": [
          {
            "address": "192.168.100.10/24",
            "gateway": "192.168.100.1"
          }
        ],
        "routes": [
          {
            "dst": "192.168.100.0/24"
          }
        ]
      }
    }
```

This approach creates a unique NAD per monitor with a hardcoded IP address.

## Practical Static IP Strategy

For a three-monitor setup with six OSDs, create a NAD for each monitor:

```bash
# Create NAD for mon-a (192.168.100.10)
cat <<EOF | kubectl apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rook-mon-a
  namespace: rook-ceph
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "static",
        "addresses": [{"address": "192.168.100.10/24", "gateway": "192.168.100.1"}]
      }
    }
EOF
```

Repeat for mon-b (192.168.100.11) and mon-c (192.168.100.12).

For OSDs (which do not need static IPs), use a shared Whereabouts-based NAD for the cluster network.

## Annotating Monitor Pods

To associate specific static-IP NADs with specific monitor pods, use pod annotations. In Rook, you can specify per-daemon annotations in the CephCluster CR:

```yaml
spec:
  mon:
    count: 3
    annotations:
      mon:
        k8s.v1.cni.cncf.io/networks: rook-ceph/rook-public-network
```

However, assigning different NADs to different mon pods requires a more advanced approach using Rook's `placement` configuration combined with pod annotations per mon ID. Check the Rook documentation for the current supported method in your version.

## Alternative: Static IPs via Node-Level Configuration

A simpler approach for predictable addressing is to configure static IPs at the node level (on the host NIC) and use Macvlan with Whereabouts that maps to that subnet:

```bash
# On each node, assign static IPs to the storage NIC
# node-1: eth1 gets 192.168.100.1/24
# node-2: eth1 gets 192.168.100.2/24
# node-3: eth1 gets 192.168.100.3/24
```

Then configure the NAD to use those node IPs directly:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rook-public-network
  namespace: rook-ceph
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "passthru",
      "ipam": {
        "type": "static"
      }
    }
```

In `passthru` mode, Macvlan passes the host interface directly to the pod, giving the pod access to the host's static IP.

## Verifying Static IP Assignment

Test with a pod to verify static IP assignment works:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-ip-test
  namespace: rook-ceph
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [{
        "name": "rook-mon-a",
        "namespace": "rook-ceph"
      }]
spec:
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
```

```bash
kubectl apply -f static-ip-test.yaml
kubectl -n rook-ceph exec static-ip-test -- ip addr show net1
```

Expected output showing the static IP:

```text
3: net1: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.100.10/24 brd 192.168.100.255 scope global net1
```

## IP Assignment Documentation

Maintain a record of static IP assignments:

```text
Monitor IP Assignments:
  mon-a: 192.168.100.10 (NAD: rook-mon-a)
  mon-b: 192.168.100.11 (NAD: rook-mon-b)
  mon-c: 192.168.100.12 (NAD: rook-mon-c)

OSD Network:
  Pool: 192.168.200.10 - 192.168.200.200 (Whereabouts)

Cluster Network CIDR: 192.168.200.0/24
Public Network CIDR: 192.168.100.0/24
```

Store this documentation alongside your Rook configuration in Git.

## Summary

Static IPs for Rook-Ceph Multus networking are configured by creating NADs with `"type": "static"` IPAM containing explicit address assignments, or by using Macvlan in passthru mode to expose host-level static IPs to pods. Create separate NADs for each monitor with its pre-assigned IP to ensure monitor IP stability. For OSDs, use Whereabouts with a shared NAD since OSD IPs do not need to be stable across pod restarts. Document all static IP assignments in your configuration repository.
