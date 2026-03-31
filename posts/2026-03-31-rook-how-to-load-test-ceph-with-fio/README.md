# How to Load Test Ceph with fio

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Performance, Benchmarking, FIO

Description: Use fio to benchmark Ceph RBD and CephFS performance with realistic workload patterns including random read/write IOPS and sequential throughput.

---

## Why Use fio for Ceph Benchmarking

While `rados bench` tests raw RADOS performance, `fio` (Flexible I/O Tester) tests performance through the RBD or CephFS layer, which is closer to how applications actually interact with storage. It supports complex workload patterns and is the industry standard for storage benchmarking.

## Setting Up fio in Kubernetes

Create a PVC for benchmarking:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fio-benchmark-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: rook-ceph-block
```

Deploy a fio pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fio-benchmark
  namespace: default
spec:
  containers:
    - name: fio
      image: nixery.dev/shell/fio
      command: ["sleep", "infinity"]
      volumeMounts:
        - mountPath: /data
          name: bench-volume
  volumes:
    - name: bench-volume
      persistentVolumeClaim:
        claimName: fio-benchmark-pvc
```

```bash
kubectl apply -f fio-pod.yaml
kubectl wait --for=condition=Ready pod/fio-benchmark --timeout=60s
```

## Random Write IOPS Test

```bash
kubectl exec -it fio-benchmark -- fio \
  --name=randwrite \
  --ioengine=libaio \
  --iodepth=16 \
  --rw=randwrite \
  --bs=4k \
  --direct=1 \
  --size=10G \
  --numjobs=4 \
  --runtime=60 \
  --group_reporting \
  --filename=/data/testfile
```

Example output:

```text
randwrite: (groupid=0, jobs=4): err= 0: pid=12345:
  write: IOPS=8500, BW=33.2MiB/s (34.8MB/s)(1993MiB/60012msec)
    clat (usec): min=318, max=52163, avg=7520.52
    lat (usec): min=319, max=52165, avg=7521.89
```

## Random Read IOPS Test

```bash
kubectl exec -it fio-benchmark -- fio \
  --name=randread \
  --ioengine=libaio \
  --iodepth=16 \
  --rw=randread \
  --bs=4k \
  --direct=1 \
  --size=10G \
  --numjobs=4 \
  --runtime=60 \
  --group_reporting \
  --filename=/data/testfile
```

## Sequential Throughput Test

```bash
kubectl exec -it fio-benchmark -- fio \
  --name=seqwrite \
  --ioengine=libaio \
  --iodepth=32 \
  --rw=write \
  --bs=1M \
  --direct=1 \
  --size=20G \
  --numjobs=2 \
  --runtime=60 \
  --group_reporting \
  --filename=/data/testfile
```

## Mixed Read/Write Workload

Simulate a database-like 70/30 read/write mix:

```bash
kubectl exec -it fio-benchmark -- fio \
  --name=mixed \
  --ioengine=libaio \
  --iodepth=16 \
  --rw=randrw \
  --rwmixread=70 \
  --bs=4k \
  --direct=1 \
  --size=10G \
  --numjobs=4 \
  --runtime=60 \
  --group_reporting \
  --filename=/data/testfile
```

## Using a fio Job File

For reproducible benchmarks, use a job file:

```ini
[global]
ioengine=libaio
direct=1
group_reporting=1
time_based=1
runtime=60

[randwrite-4k]
name=randwrite-4k
rw=randwrite
bs=4k
iodepth=16
numjobs=4
filename=/data/rw4k

[randread-4k]
name=randread-4k
rw=randread
bs=4k
iodepth=16
numjobs=4
filename=/data/rr4k

[seqwrite-1m]
name=seqwrite-1m
rw=write
bs=1M
iodepth=32
numjobs=2
filename=/data/sw1m
```

Run the job file:

```bash
kubectl cp fio-jobs.ini fio-benchmark:/tmp/fio-jobs.ini
kubectl exec -it fio-benchmark -- fio /tmp/fio-jobs.ini
```

## Key Metrics to Capture

- **IOPS** - operations per second at 4K random (database workloads)
- **Bandwidth** - MB/s at 1M sequential (backup, streaming)
- **Latency** - average and p99 latency at your workload's block size
- **Queue depth sensitivity** - IOPS at iodepth=1 vs iodepth=32

## Cleanup

```bash
kubectl delete pod fio-benchmark
kubectl delete pvc fio-benchmark-pvc
```

## Summary

fio provides comprehensive Ceph storage benchmarking through the full RBD or CephFS stack. Run separate tests for random 4K IOPS, sequential 1M throughput, and mixed read/write workloads to understand how your cluster will perform for different application types. Use job files to create reproducible benchmark suites for regression testing after cluster changes.
