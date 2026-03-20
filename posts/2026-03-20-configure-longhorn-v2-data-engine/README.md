# How to Configure Longhorn V2 Data Engine

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, V2 Data Engine, SPDK, NVMe, Performance

Description: Configure and enable Longhorn's V2 Data Engine based on SPDK for ultra-high performance storage with significantly lower CPU overhead and latency.

## Introduction

Longhorn's V2 Data Engine (introduced in Longhorn v1.6.0) is a new storage engine based on SPDK (Storage Performance Development Kit). Unlike the V1 engine which uses kernel-space block devices, the V2 engine uses user-space NVMe over Fabrics to achieve near-bare-metal NVMe performance with drastically lower CPU utilization. This guide explains how to configure and use the V2 Data Engine.

## V1 vs V2 Data Engine Comparison

| Feature | V1 Engine | V2 Engine |
|---------|-----------|-----------|
| Storage subsystem | Kernel (tgt) | User-space (SPDK) |
| Protocol | iSCSI | NVMe-oF TCP |
| Typical IOPS | 100K-500K | 1M+ |
| CPU efficiency | Standard | Much lower CPU per IOPS |
| Latency | ~100-500μs | ~50-100μs |
| Stability | Production | Beta (as of v1.7) |
| NVMe requirement | No | Recommended |

## Prerequisites

- Longhorn v1.6.0 or later
- Linux kernel 5.15 or later with NVMe-oF TCP support
- At least 2 CPU cores dedicated to SPDK (hugepages required)
- NVMe SSDs recommended (though not strictly required)

### Verify Kernel Support

```bash
# Check kernel version (need 5.15+)

uname -r

# Verify NVMe-oF TCP module
modprobe nvme-tcp
lsmod | grep nvme_tcp

# Load module permanently
echo "nvme-tcp" >> /etc/modules
```

### Configure Hugepages

SPDK requires hugepages for its memory allocator:

```bash
# Configure 2 GiB of hugepages per node (minimum for SPDK)
# Add to /etc/sysctl.conf for persistence
echo "vm.nr_hugepages = 1024" >> /etc/sysctl.conf
sysctl -p

# Verify hugepages
cat /proc/meminfo | grep HugePages
# HugePages_Total: 1024
# HugePages_Free: 1024

# Or configure via Kubernetes (requires node restart or DaemonSet)
```

## Enable V2 Data Engine in Longhorn

### Step 1: Enable the V2 Data Engine Feature

```bash
# Enable the V2 data engine in Longhorn settings
kubectl patch settings.longhorn.io v2-data-engine \
  -n longhorn-system \
  --type merge \
  -p '{"value": "true"}'

# Wait for V2 engine pods to be created
kubectl get pods -n longhorn-system | grep instance-manager
```

### Step 2: Configure Node-Level V2 Settings

The V2 engine requires configuring the number of hugepages and CPU cores per node:

```bash
# Patch a Longhorn node to configure V2 engine settings
kubectl patch nodes.longhorn.io worker-node-1 \
  -n longhorn-system \
  --type merge \
  -p '{
    "spec": {
      "instanceManagerCPURequest": 250,
      "instanceManagerCPULimit": 1000
    }
  }'
```

### Step 3: Set Hugepage Configuration

```yaml
# longhorn-node-v2.yaml - Configure hugepages for V2 engine
apiVersion: longhorn.io/v1beta2
kind: Node
metadata:
  name: worker-node-1
  namespace: longhorn-system
spec:
  allowScheduling: true
  # Hugepages allocated for SPDK (in MiB)
  hugepageRequestForV2DataEngine: 2048  # 2 GiB
```

```bash
kubectl apply -f longhorn-node-v2.yaml
```

## Create a V2 StorageClass

```yaml
# storageclass-v2.yaml - StorageClass using the V2 data engine
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-v2
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "3"
  dataLocality: "best-effort"
  fsType: "ext4"
  # Enable V2 data engine for this storage class
  dataEngine: "v2"
  # Recommended disk selector for NVMe disks
  diskSelector: "nvme"
```

```bash
kubectl apply -f storageclass-v2.yaml
```

## Deploy a Test Workload with V2 Engine

```yaml
# test-v2-pvc.yaml - PVC using V2 data engine
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: v2-engine-test
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-v2
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: v2-test-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: v2-engine-test
```

```bash
kubectl apply -f test-v2-pvc.yaml

# Verify the V2 engine is being used
kubectl describe volume.longhorn.io <volume-name> -n longhorn-system | grep "Data Engine"
```

## Benchmarking V2 vs V1 Performance

```bash
# Compare performance between V1 and V2 engines
# First, create benchmark PVCs for each engine type

# Benchmark V2 engine
kubectl exec -it v2-test-pod -- \
  fio --name=v2-bench \
    --rw=randread \
    --bs=4k \
    --size=1g \
    --filename=/data/bench \
    --ioengine=libaio \
    --iodepth=128 \
    --direct=1 \
    --numjobs=4 \
    --output-format=json 2>&1 | tee v2-results.json
```

## Monitoring V2 Engine Metrics

```bash
# Check V2 instance manager pods
kubectl get pods -n longhorn-system | grep instance-manager-e-v2

# View V2 engine specific metrics
kubectl logs -n longhorn-system \
  $(kubectl get pods -n longhorn-system \
    -l longhorn.io/instance-manager-type=engine \
    -l longhorn.io/data-engine=v2 \
    -o name | head -1) \
  --tail=50
```

## Limitations of V2 Data Engine (Beta)

As of Longhorn v1.7, the V2 engine has some limitations:
- No support for RWX (ReadWriteMany) volumes
- No volume encryption support yet
- Snapshots work differently (snapshot is a point-in-time copy of the device)
- Backup to external targets may have limitations

## Conclusion

Longhorn's V2 Data Engine represents a significant leap in storage performance for latency-sensitive applications. By leveraging SPDK's user-space NVMe processing, it achieves much higher IOPS with lower CPU overhead compared to the V1 engine. While still in beta, the V2 engine is worth evaluating for performance-critical workloads. Monitor its development in Longhorn releases, and always test thoroughly before adopting it for production workloads.
