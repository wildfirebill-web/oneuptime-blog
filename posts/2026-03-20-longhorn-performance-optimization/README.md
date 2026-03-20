# How to Optimize Longhorn Performance for Production

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Performance, Optimization, Kubernetes, Storage, Production, SUSE Rancher

Description: Learn how to optimize Longhorn storage performance for production workloads by tuning replica counts, network settings, disk configurations, and CPU resource allocation.

---

Longhorn's default settings prioritize safety and ease of use over raw performance. For production workloads, targeted tuning can significantly improve throughput and reduce latency.

---

## Performance Baseline

Before tuning, establish a baseline:

```bash
# Install fio for storage benchmarking
kubectl run fio-test --image=xridge/fio:latest --restart=Never \
  --overrides='{"spec":{"volumes":[{"name":"test","persistentVolumeClaim":{"claimName":"test-pvc"}}],"containers":[{"name":"fio","image":"xridge/fio:latest","command":["sleep","infinity"],"volumeMounts":[{"name":"test","mountPath":"/data"}]}]}}'

# Run sequential write test
kubectl exec fio-test -- fio \
  --name=seq-write \
  --rw=write \
  --bs=1M \
  --size=4G \
  --numjobs=4 \
  --ioengine=libaio \
  --iodepth=32 \
  --runtime=60 \
  --filename=/data/test \
  --output-format=json

# Run random read/write test (IOPS)
kubectl exec fio-test -- fio \
  --name=rand-rw \
  --rw=randrw \
  --bs=4k \
  --size=1G \
  --numjobs=4 \
  --ioengine=libaio \
  --iodepth=64 \
  --runtime=60 \
  --filename=/data/test
```

---

## Optimization 1: Reduce Replica Count for Non-Critical Data

```yaml
# High-performance StorageClass with 2 replicas
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-perf
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "2"          # Reduce from 3 — less write amplification
  staleReplicaTimeout: "20"
  dataLocality: "best-effort"    # Prefer local replica reads
  diskSelector: ""
  nodeSelector: ""
```

---

## Optimization 2: Enable Data Locality

Data locality reduces latency by ensuring at least one replica is on the same node as the workload:

```bash
# Set global data locality to "best-effort"
kubectl patch setting -n longhorn-system \
  default-data-locality \
  --type merge \
  -p '{"value":"best-effort"}'

# Or configure per volume in StorageClass
# dataLocality: "strict-local"   # Always use local replica (fastest)
# dataLocality: "best-effort"    # Prefer local, fall back to remote
# dataLocality: "disabled"       # Default — no preference
```

---

## Optimization 3: Use Dedicated Disks for Longhorn

```bash
# In the Longhorn UI: Node → Edit → Add a dedicated disk
# Set the disk path to a dedicated SSD mount point

# On the node, mount a dedicated disk:
# /etc/fstab entry:
# /dev/nvme1n1  /var/lib/longhorn-fast  ext4  defaults  0  2

# Then in Longhorn, add the disk with path: /var/lib/longhorn-fast
# and add a tag: "ssd" or "fast"
```

---

## Optimization 4: Tune CPU Resources for Instance Managers

```bash
# Set CPU and memory requests for instance manager pods
kubectl patch setting -n longhorn-system \
  guaranteed-instance-manager-cpu \
  --type merge \
  -p '{"value":"250"}'    # 250m = 0.25 CPU cores per instance manager

# For high-I/O workloads, increase this value
kubectl patch setting -n longhorn-system \
  guaranteed-instance-manager-cpu \
  --type merge \
  -p '{"value":"500"}'
```

---

## Optimization 5: Configure Replica Auto-Balance

```bash
# Enable replica auto-balance to keep replicas evenly distributed
kubectl patch setting -n longhorn-system \
  replica-auto-balance \
  --type merge \
  -p '{"value":"best-effort"}'
```

---

## Optimization 6: Disable Revision Counter for Read-Heavy Workloads

The revision counter writes metadata on every I/O operation. Disabling it reduces write amplification:

```bash
# Disable revision counter
kubectl patch setting -n longhorn-system \
  disable-revision-counter \
  --type merge \
  -p '{"value":"true"}'
```

Note: Disabling revision counter means Longhorn cannot detect dirty replicas after an unclean shutdown — enable only for workloads with application-level consistency guarantees.

---

## Optimization 7: Network Tuning

Longhorn replication uses TCP. On high-latency or high-bandwidth networks:

```bash
# Increase network buffer sizes on nodes (via DaemonSet or node configuration)
sysctl -w net.core.rmem_max=268435456
sysctl -w net.core.wmem_max=268435456
sysctl -w net.ipv4.tcp_rmem="4096 87380 268435456"
sysctl -w net.ipv4.tcp_wmem="4096 65536 268435456"
```

---

## Performance Summary

| Setting | Default | Optimized |
|---|---|---|
| Replica count | 3 | 2 (non-critical data) |
| Data locality | disabled | best-effort |
| Instance manager CPU | 12m | 250-500m |
| Revision counter | enabled | disabled (with caution) |
| Replica auto-balance | disabled | best-effort |

---

## Best Practices

- Apply `dataLocality: strict-local` only for stateful workloads with node affinity — otherwise pod migrations will cause remote I/O.
- Benchmark after each change to verify the improvement — not all optimizations help equally for all workload patterns.
- Use dedicated SSDs for Longhorn data directories on database nodes and keep spinning disks for backup or archival volumes.
