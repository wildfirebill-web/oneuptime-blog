# How to Set MDS Active Count and Active Standby in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, MDS, CephFS, Metadata Server, High Availability, Kubernetes

Description: Configure MDS activeCount and activeStandby settings in Rook to control CephFS metadata server availability, failover speed, and multi-rank scalability.

---

## What Are MDS Daemons

The Metadata Server (MDS) is the Ceph daemon responsible for managing CephFS directory trees, file metadata, and namespace operations. Every CephFS filesystem requires at least one active MDS daemon. Rook manages MDS deployment through the `metadataServer` section of the CephFilesystem CRD.

## Key MDS Configuration Fields

Two critical fields control MDS topology:

- `activeCount`: number of simultaneously active MDS daemons (ranks). More active ranks enable directory-level parallelism for very large filesystems.
- `activeStandby`: when `true`, Rook deploys extra standby MDS daemons that warm their caches and are ready to take over a failed active rank immediately (warm standby). When `false`, only cold standbys are deployed.

## Single Active MDS with Warm Standby

The standard production configuration uses one active rank and one warm standby:

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
    - name: replicated
      failureDomain: host
      replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
```

With `activeStandby: true`, Rook deploys `activeCount * 2` MDS pods total. For `activeCount: 1`, you get one active and one standby. Failover from active to standby takes seconds because the standby has already loaded the MDS journal.

## Multiple Active MDS Ranks

For filesystems with millions of files or hundreds of simultaneous clients, multiple active ranks distribute the metadata load. Each active rank owns a subtree of the directory hierarchy:

```yaml
  metadataServer:
    activeCount: 2
    activeStandby: true
```

With `activeCount: 2` and `activeStandby: true`, Rook deploys four MDS pods: two active and two standbys. Verify the rank assignments:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs status myfs
```

The output shows rank assignments and which daemon holds each rank.

## Scaling Active MDS Count

You can increase `activeCount` without downtime. Rook will deploy additional MDS pods and Ceph will rebalance the directory subtrees:

```bash
kubectl -n rook-ceph edit cephfilesystem myfs
# Change activeCount from 1 to 2
```

Monitor the state change:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch ceph fs status myfs
```

Wait until all new ranks show `active` before considering the change complete.

## Resource Requests for MDS Pods

When increasing `activeCount`, also set appropriate resource requests so MDS pods are scheduled on nodes with sufficient memory:

```yaml
  metadataServer:
    activeCount: 2
    activeStandby: true
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
```

MDS memory consumption scales with the number of cached inodes. Monitor MDS memory usage:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.myfs-a heap stats
```

## Placing MDS Pods on Specific Nodes

Use placement configurations to keep MDS pods off OSD nodes:

```yaml
  metadataServer:
    activeCount: 1
    activeStandby: true
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

## Verifying MDS Health

Check that all expected MDS daemons are running and healthy:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mds

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mds stat
```

The `ceph mds stat` output should show the correct number of active and standby daemons with no daemons in `failed` or `damaged` state.

## Summary

The `activeCount` field in the Rook CephFilesystem `metadataServer` spec controls how many MDS ranks handle metadata concurrently, and `activeStandby: true` ensures warm standbys are always ready to replace a failed active rank without full cache cold-start. Start with `activeCount: 1` and increase it only for very large filesystems with demonstrated MDS bottlenecks.
