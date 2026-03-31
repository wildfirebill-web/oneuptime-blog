# How to Set Up Multiple Active MDS for CephFS Scalability

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, MDS, CephFS, Scalability, Kubernetes, Performance

Description: Learn how to configure multiple active MDS daemons for CephFS to scale metadata throughput horizontally across large or metadata-intensive filesystems.

---

## When to Use Multiple Active MDS

A single active MDS can handle roughly 1,000-5,000 metadata operations per second. If your workload involves thousands of concurrent clients, large numbers of small files, or intensive directory traversal, a single MDS becomes a bottleneck. CephFS supports multiple active MDS daemons through directory subtree partitioning - each MDS owns a portion of the directory tree.

## Configuring Multiple Active MDS in Rook

Update the CephFilesystem spec to increase the `activeCount`:

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
  - replicated:
      size: 3
  metadataServer:
    activeCount: 3
    activeStandby: true
    resources:
      requests:
        memory: "8Gi"
        cpu: "2"
      limits:
        memory: "16Gi"
        cpu: "4"
```

Apply the change:

```bash
kubectl apply -f filesystem.yaml
```

## Verifying Multiple Active MDS

Check that the desired number of MDS daemons are active:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mds

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs status myfs
```

The output should show `3 up:active` with additional standby daemons.

## Understanding Subtree Pinning

With multiple active MDS, the MDS balancer distributes directory subtrees across daemons. Clients automatically connect to the correct MDS for their working directory. Check the current distribution:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mds stat
```

## Manual Subtree Pinning for Predictable Distribution

For workloads where you know the access pattern, pin specific directories to specific MDS ranks:

```bash
# Pin /data/tenant-a to MDS rank 0
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c \
  "setfattr -n ceph.dir.pin -v 0 /mnt/myfs/data/tenant-a"

# Pin /data/tenant-b to MDS rank 1
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c \
  "setfattr -n ceph.dir.pin -v 1 /mnt/myfs/data/tenant-b"
```

## Configuring the MDS Balancer

The automatic balancer distributes load across active MDS daemons:

```bash
# Enable the balancer
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_bal_mode 2

# Set the interval for rebalancing (seconds)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_bal_interval 10
```

## Scaling Considerations

Important notes when running multiple active MDS:

- Each active MDS needs its own metadata pool journal segment
- The metadata pool must have sufficient PGs to handle increased load
- Network bandwidth between MDS daemons increases with active count
- Standby count should be at least equal to active count for HA

```bash
# Ensure enough journal segments are configured
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_journal_max_events 100000
```

## Summary

Multiple active MDS daemons horizontally scale CephFS metadata throughput by partitioning the directory namespace across daemons. Configure `activeCount` in the CephFilesystem CRD, verify all daemons reach active state, and use subtree pinning or the automatic balancer to distribute load. For predictable high-throughput workloads, manual directory pinning ensures consistent performance without balancer overhead.
