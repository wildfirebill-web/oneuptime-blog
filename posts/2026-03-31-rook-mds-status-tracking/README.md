# How to Check MDS Status and Active/Inactive Tracking in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, MDS, CephFS, Monitoring

Description: Monitor CephFS MDS daemon status including active, standby, and standby-replay states, and track MDS failover events in Rook-managed clusters.

---

The Metadata Server (MDS) manages the CephFS filesystem namespace. For a healthy CephFS deployment, you need at least one active MDS and typically one standby MDS for high availability. Monitoring MDS status helps detect failover events and slow metadata operations.

## Check MDS Status

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph mds stat
```

Sample output:

```text
myfs:1 {0=myfs-b=up:active} 1 up:standby
```

This shows:
- Filesystem `myfs` with 1 active MDS (`myfs-b`) and 1 standby

## Get Detailed MDS Information

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph fs status myfs
```

Sample output:

```text
myfs - 5 clients
=======
RANK  STATE      MDS          ACTIVITY     DNS    INOS   DIRS   CAPS
 0    active     myfs-b       Reqs:    1/s  5k    10k    500    2k

POOL        TYPE     USED  AVAIL
myfs-metadata  metadata  1GiB  5TiB
myfs-data0     data      100GiB  5TiB

STANDBY MDS
myfs-a
```

## MDS States Explained

| State | Meaning |
|---|---|
| `up:active` | Serving metadata I/O requests |
| `up:standby` | Idle, ready to take over |
| `up:standby-replay` | Replaying active MDS journal for fast failover |
| `up:creating` | Initializing new filesystem |
| `up:rejoin` | Re-joining after recovery |
| `up:stopping` | Gracefully shutting down |

## Configure Standby MDS in Rook

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
  - name: data0
    replicated:
      size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true   # enables standby-replay
```

With `activeStandby: true`, the standby MDS replays the active journal in real-time, enabling sub-second failover.

## Check MDS Pods in Kubernetes

```bash
kubectl get pods -n rook-ceph -l app=rook-ceph-mds
```

Expected output:

```text
NAME                              READY   STATUS    RESTARTS   AGE
rook-ceph-mds-myfs-a-...          2/2     Running   0          5d
rook-ceph-mds-myfs-b-...          2/2     Running   0          5d
```

## Monitor MDS Performance

```bash
# Check current metadata operation rates
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph fs status myfs | grep -A2 "RANK"

# Check MDS cache size
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph tell mds.myfs-b cache status
```

## Check MDS Slow Requests

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph tell mds.myfs-b dump_ops_in_flight
```

If you see slow ops, check MDS memory and CPU resources.

## View MDS Failover History

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph log last 100 | grep -i mds
```

## Scale Active MDS Count

For multiple active MDS instances (sharded metadata):

```yaml
spec:
  metadataServer:
    activeCount: 2   # Shard metadata across 2 active MDS
    activeStandby: true
```

This requires at least 4 MDS pods (2 active + 2 standby).

## Summary

CephFS MDS status is checked with `ceph mds stat` for a quick overview and `ceph fs status <fs-name>` for detailed state including pool usage and client counts. Configure `activeStandby: true` in Rook's `CephFilesystem` CRD for fast failover. Monitor for `slow requests` in MDS logs and ensure adequate memory allocation since MDS performance is heavily memory-dependent.
