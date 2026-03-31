# How to Troubleshoot Known Multus Limitations with Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Multus, Kubernetes, Network, Troubleshooting

Description: Learn how to identify and work around known Multus CNI limitations when running Rook-Ceph clusters in Kubernetes environments.

---

## Overview

Multus CNI enables Rook-Ceph pods to use dedicated network interfaces for storage traffic, separating it from the cluster's default network. However, Multus has known limitations that can cause subtle failures in Rook deployments. Understanding these constraints helps you design resilient storage networks and diagnose issues faster.

## Known Limitation: OSD Pods and Network Attachment

One of the most common issues is that OSD pods fail to attach to a `NetworkAttachmentDefinition` when the referenced interface does not exist on the node. This typically happens when OSDs are scheduled on nodes where the secondary NIC is not yet configured.

```bash
kubectl describe pod rook-ceph-osd-0-xxx -n rook-ceph | grep -A 10 Events
```

A typical error looks like:

```text
Failed to create pod sandbox: failed to setup network for sandbox: error adding pod to network "macvlan-conf": failed to find interface "eth1"
```

Fix this by adding node affinity or node selectors to ensure OSDs only land on properly prepared nodes:

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
      public: rook-ceph/public-net
      cluster: rook-ceph/cluster-net
  placement:
    osd:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
                - key: multus-enabled
                  operator: In
                  values:
                    - "true"
```

## Known Limitation: Monitor Pods Cannot Use Multus

Ceph Monitor pods currently do not support Multus network selection. Monitors always use the default pod network. This means monitor traffic is not isolated even when OSDs use dedicated interfaces.

Verify monitor network configuration:

```bash
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph mon dump
```

The output will show monitor addresses bound to the default pod CIDR, not your Multus-attached addresses. Plan your firewall and bandwidth allocation accordingly.

## Known Limitation: Readiness Probes and Secondary IPs

Kubernetes readiness probes always use the primary pod IP. When Multus attaches a secondary interface, health checks still target the default network IP. This can cause pods to appear unhealthy if your Ceph daemons attempt to bind exclusively to the Multus interface.

Check which address a daemon is listening on:

```bash
kubectl exec -it rook-ceph-osd-0-xxx -n rook-ceph -- ss -tnlp | grep ceph
```

If the daemon binds only to the Multus IP, the kubelet readiness probe will fail. Ensure daemons bind to `0.0.0.0` or include the primary IP in their bind address list.

## Known Limitation: IPAM Exhaustion

Multus relies on the underlying IPAM plugin (e.g., whereabouts, host-local) to allocate IPs for secondary interfaces. In large clusters, whereabouts can suffer from IP allocation races or exhausted ranges.

Check whereabouts lease state:

```bash
kubectl get ippools -n rook-ceph
kubectl describe ippool <pool-name> -n rook-ceph
```

Increase the IP range or use a larger subnet in your `NetworkAttachmentDefinition`:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: public-net
  namespace: rook-ceph
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.0/22"
      }
    }
```

## Debugging with Logs

Collect Multus logs directly from the node:

```bash
journalctl -u kubelet --since "10 minutes ago" | grep -i multus
```

Also inspect CNI logs on the node:

```bash
ls /var/log/pods/rook-ceph_*/
cat /var/log/pods/rook-ceph_rook-ceph-osd-0-xxx/rook-ceph-osd/0.log
```

## Summary

Multus limitations in Rook-Ceph deployments commonly involve OSD scheduling on unprepared nodes, monitor pods being excluded from secondary network selection, readiness probe mismatches with secondary IPs, and IPAM pool exhaustion. Addressing these requires careful node labeling, correctly scoped NetworkAttachmentDefinitions with adequate IP ranges, and ensuring daemon bind addresses cover the primary pod IP for health checks.
