# How to Optimize CephFS for Random Access Patterns

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Performance, Optimization

Description: Optimize CephFS in Rook for random access workloads by tuning OSD caching, object layout, and client settings to reduce latency for IOPS-intensive applications.

---

## Random Access Workload Characteristics

Random access patterns are common in databases, virtual machine disks accessed via CephFS, and analytics workloads that query arbitrary file offsets. Unlike sequential workloads, they generate many small I/O operations at unpredictable positions. CephFS latency per operation matters more than aggregate bandwidth.

## SSD OSD Placement

For random access, SSD OSDs are essential. Configure the pool to use only SSD devices:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: randomfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
      deviceClass: ssd
  dataPools:
    - name: ssd-data
      replicated:
        size: 3
        deviceClass: ssd
  metadataServer:
    activeCount: 1
    activeStandby: true
```

## BlueStore Cache Tuning

Maximize BlueStore's ability to serve reads from cache:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd bluestore_cache_size_ssd 8589934592

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd bluestore_cache_kv_ratio 0.2

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd bluestore_cache_data_ratio 0.7
```

## Object Size Alignment

For random workloads with small I/O (4 KB to 64 KB), reduce the object size to minimize write amplification:

```bash
setfattr -n ceph.dir.layout.object_size -v 4194304 /mnt/cephfs/random-data
setfattr -n ceph.dir.layout.stripe_count -v 1 /mnt/cephfs/random-data
```

Smaller objects mean more RADOS objects per file, which spreads I/O across more OSDs in parallel.

## Disable Readahead for Random Access

Readahead is counterproductive for random workloads - it wastes bandwidth fetching data that will not be used:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client client_readahead_max_bytes 0
```

Or per-mount via mount options in the StorageClass:

```yaml
mountOptions:
  - readdir_max_bytes=1048576
  - dcache_timeout=10
```

## MDS Tuning for Random Metadata Access

Random access workloads often involve frequent `stat` calls. Increase the MDS cache to keep more inodes in memory:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_cache_memory_limit 4294967296
```

Set a shorter MDS beacon interval to detect and recover from MDS issues faster:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_beacon_interval 4
```

## OSD Operation Scheduler

Switch to the mclock scheduler for more predictable latency:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_op_queue mclock_scheduler
```

## Benchmarking Random IOPS

Test random 4K read IOPS:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  fio --name=rand-read --ioengine=libaio --rw=randread \
      --bs=4k --size=4g --iodepth=32 --numjobs=4 \
      --filename=/mnt/cephfs/bench/rand-test
```

Compare average latency with `--output-format=json` and look for `lat_ns.mean`.

## Summary

Random access optimization in CephFS requires SSD OSDs, a large BlueStore cache, and disabling readahead so bandwidth is not wasted on prefetch. Using smaller object sizes improves OSD parallelism for small I/O operations. Switching to the mclock OSD scheduler reduces tail latency under concurrent workloads.
