# How to Configure Active-Active MDS Servers in Rook CephFilesystem

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CephFilesystem, MDS, Performance

Description: Scale CephFS metadata performance by configuring multiple active MDS servers in Rook using filesystem subtree pinning and activeCount settings.

---

By default, CephFS uses a single active MDS server. For large-scale deployments with heavy metadata workloads, you can configure multiple active MDS servers to distribute metadata across different subtrees of the filesystem hierarchy.

## Single vs Multi-Active MDS

```mermaid
flowchart TD
    subgraph Single Active
        A[MDS Active] --> B[All metadata]
        C[MDS Standby] --> D[Hot standby only]
    end
    subgraph Multi-Active
        E[MDS Active 0] --> F[/data/team-a subtree]
        G[MDS Active 1] --> H[/data/team-b subtree]
        I[MDS Active 2] --> J[/data/team-c subtree]
        K[MDS Standby] --> L[Failover for any active]
    end
```

## Configure activeCount in CephFilesystem

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    failureDomain: host
    replicated:
      size: 3
  dataPools:
    - name: data0
      failureDomain: host
      replicated:
        size: 3
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 3        # three active MDS servers
    activeStandby: true   # deploy standby for each active
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
      limits:
        cpu: "4"
        memory: "8Gi"
    placement:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - rook-ceph-mds
            topologyKey: kubernetes.io/hostname
```

This creates 6 MDS pods total: 3 active and 3 standby (when `activeStandby: true`).

## Verify MDS Deployment

```bash
# Check MDS pods - should see 6 pods for activeCount: 3 with standby
kubectl get pods -n rook-ceph -l app=rook-ceph-mds

# Check MDS status in Ceph
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph fs status myfs
```

Example output:

```text
myfs - 3 clients
========
RANK   STATE            MDS                 ACTIVITY
   0   active           myfs-a-58d7fb-xxx   Reqs:  42 /s
   1   active           myfs-b-6c9d5c-xxx   Reqs:  38 /s
   2   active           myfs-c-7e2f1a-xxx   Reqs:  35 /s
0-s   standby-replay   myfs-a-standby-xxx
1-s   standby-replay   myfs-b-standby-xxx
2-s   standby-replay   myfs-c-standby-xxx
```

## Subtree Pinning for Workload Distribution

With multiple active MDS servers, you must pin directory subtrees to specific MDS ranks to distribute the metadata load:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- bash

# Mount CephFS temporarily for pinning
# (or use the ceph fs subvolume set-pin command)

# Pin /volumes/team-a to MDS rank 0
ceph fs subvolume pin myfs team-a export 0

# Pin /volumes/team-b to MDS rank 1
ceph fs subvolume pin myfs team-b export 1

# Pin /volumes/team-c to MDS rank 2
ceph fs subvolume pin myfs team-c export 2
```

Using xattr-based pinning directly on directories (requires mounting CephFS):

```bash
# Set the MDS rank for a directory via xattr
setfattr -n ceph.dir.pin -v 0 /mnt/cephfs/team-a
setfattr -n ceph.dir.pin -v 1 /mnt/cephfs/team-b
setfattr -n ceph.dir.pin -v 2 /mnt/cephfs/team-c
```

## Automatic Subtree Pinning

Alternatively, enable automatic load balancing (experimental):

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mds mds_bal_export_pin -1
```

Setting the pin value to `-1` allows the balancer to choose placement automatically.

## MDS Cache Tuning

Each active MDS uses a metadata cache. Tune its size for your workload:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- bash

# Set per-MDS cache size (in bytes)
ceph config set mds mds_cache_memory_limit 4294967296   # 4 GiB

# Alternatively, set in the CephFilesystem CR
```

```yaml
spec:
  metadataServer:
    activeCount: 3
    activeStandby: true
    resources:
      limits:
        memory: "8Gi"   # Kubernetes limit - must be higher than mds_cache_memory_limit
```

## Monitor MDS Performance

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- bash

# Check request rates per MDS
ceph fs status myfs

# Detailed MDS stats
ceph daemon mds.myfs-a perf dump | python3 -m json.tool | grep -A5 request

# Check cache usage
ceph daemon mds.myfs-a cache status
```

## Scale Down MDS

To reduce from 3 to 1 active MDS, update the CephFilesystem:

```yaml
spec:
  metadataServer:
    activeCount: 1
    activeStandby: true
```

```bash
kubectl apply -f cephfilesystem.yaml
# Rook will gracefully reduce the active count
kubectl get pods -n rook-ceph -l app=rook-ceph-mds -w
```

## Summary

Multiple active MDS servers in Rook CephFilesystem (`activeCount > 1`) scale metadata throughput for large filesystems with heavy workloads. Pair increased `activeCount` with directory subtree pinning to distribute load across MDS ranks. Always set `activeStandby: true` for high availability, and tune the MDS cache memory to match the Kubernetes resource limits.
