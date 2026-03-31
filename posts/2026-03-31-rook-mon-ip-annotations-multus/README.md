# How to Set Mon IP Annotations with Multus in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitor, Multus, Annotation

Description: Learn how to use Rook's mon IP annotations to specify which Multus network interface addresses Ceph monitors should advertise for cluster communication.

---

When using Multus with Rook-Ceph, monitors must advertise their Multus network IP addresses (not the primary Kubernetes pod network IPs) so that other Ceph daemons and clients can reach them via the storage network. Rook provides annotations to specify which IP each monitor should use for its public address.

## The Problem with Multi-Network Pods

When a pod has multiple network interfaces (primary Kubernetes network + Multus secondary network), Ceph needs to know which IP to advertise in the monitor map. By default, Ceph might bind to the primary Kubernetes pod network IP, which defeats the purpose of the dedicated storage network.

The Ceph monitor map stores the actual IP:port where each monitor listens. If monitors advertise pod network IPs, OSDs and clients connecting from outside the cluster will fail to reach them.

Check what IPs your monitors are currently using:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon dump
```

```text
0: [v2:10.244.0.15:3300/0,v1:10.244.0.15:6789/0] mon.a
```

If you see `10.244.x.x` (typical pod network IPs) instead of your storage network IPs, monitors are not using the Multus interface.

## Setting Mon IP Annotations

Rook reads the `rook.io/mon-endpoint` annotation on monitor pods to determine the IP address to register in the monitor map. Set this annotation in the CephCluster spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  network:
    provider: multus
    selectors:
      public: rook-ceph/rook-public-network
      cluster: rook-ceph/rook-cluster-network
  mon:
    count: 3
    annotations:
      mon:
        rook.io/mon-ip: "192.168.100.10"
```

However, this annotation approach sets the same IP for all monitors. For per-monitor IP control, use the ConfigMap approach.

## Per-Monitor IP Configuration via ConfigMap

Rook allows specifying per-monitor IPs through a ConfigMap that the operator reads during mon creation:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-ceph-mon-endpoints
  namespace: rook-ceph
data:
  data: "a=192.168.100.10:6789,b=192.168.100.11:6789,c=192.168.100.12:6789"
  mapping: '{"node":{"a":{"Name":"node-1","Hostname":"node-1","Address":"192.168.100.10"},"b":{"Name":"node-2","Hostname":"node-2","Address":"192.168.100.11"},"c":{"Name":"node-3","Hostname":"node-3","Address":"192.168.100.12"}}}'
```

This ConfigMap pre-configures which storage network IP each monitor will use.

## Using Pod Annotations Directly

You can also annotate individual monitor pods after they start, to force them to use specific IPs:

```bash
# Get the mon pod name
MON_POD=$(kubectl -n rook-ceph get pod -l app=rook-ceph-mon,ceph_daemon_id=a -o name)

# Get the Multus-assigned IP
MULTUS_IP=$(kubectl -n rook-ceph exec $MON_POD -- ip addr show net1 | \
  grep "inet " | awk '{print $2}' | cut -d'/' -f1)

echo "Mon-a Multus IP: $MULTUS_IP"

# Annotate the pod (Rook reads this)
kubectl -n rook-ceph annotate pod $(basename $MON_POD) \
  network.operator.openshift.io/interfaces-data="${MULTUS_IP}"
```

## Verifying Monitor Addresses

After configuring mon IP annotations, verify that the monitor map uses storage network IPs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon dump
```

Expected output with Multus storage network IPs:

```text
epoch 12
fsid xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
min_mon_release 17 (quincy)
0: [v2:192.168.100.10:3300/0,v1:192.168.100.10:6789/0] mon.a
1: [v2:192.168.100.11:3300/0,v1:192.168.100.11:6789/0] mon.b
2: [v2:192.168.100.12:3300/0,v1:192.168.100.12:6789/0] mon.c
```

The IPs `192.168.100.x` are from the Multus storage network, not the Kubernetes pod network.

## Handling Mon Address Changes

If a monitor starts with a pod network IP and you later update the annotation to use the Multus IP, the mon map must be updated. This requires restarting the monitor:

```bash
# Delete the mon pod - Rook will recreate it with the correct IP
kubectl -n rook-ceph delete pod $(kubectl -n rook-ceph get pod \
  -l app=rook-ceph-mon,ceph_daemon_id=a -o name | head -1 | xargs basename)

# Watch the new pod start with the correct IP
kubectl -n rook-ceph get pod -l app=rook-ceph-mon -w
```

After restart, verify the mon map is updated:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon dump | grep "mon.a"
```

## Configuring Ceph Client Access

When monitors use Multus storage network IPs, Ceph clients (including the CSI driver and the toolbox) must be able to reach those IPs. Ensure the Rook toolbox pod also has the Multus network attached:

```yaml
spec:
  network:
    provider: multus
    selectors:
      public: rook-ceph/rook-public-network
```

With `provider: multus` set cluster-wide, Rook automatically attaches the Multus network to all Ceph-related pods including the toolbox.

## Summary

Setting mon IP annotations with Multus in Rook ensures Ceph monitors advertise their storage network IP addresses (from Multus secondary interfaces) rather than Kubernetes pod network IPs. Configure per-monitor IPs using the `rook-ceph-mon-endpoints` ConfigMap or per-pod annotations. After updating monitor IPs, restart mon pods to update the monitor map and verify the correct IPs using `ceph mon dump`. Ensure all Ceph-related pods (including toolbox and CSI driver) have the Multus storage network attached so they can reach monitors via the storage network.
