# How to Configure Longhorn V2 Data Engine

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, V2 Data Engine, SPDK, NVMe, Kubernetes, Storage, High Performance

Description: Learn how to configure Longhorn's V2 Data Engine based on SPDK to achieve significantly higher IOPS and lower latency compared to the V1 iSCSI-based engine.

---

Longhorn V2 Data Engine (introduced in Longhorn v1.5) uses SPDK (Storage Performance Development Kit) to deliver near-NVMe performance by bypassing the kernel's storage stack. This guide covers enabling and configuring V2.

---

## Prerequisites

- Longhorn v1.5.0+
- Linux kernel 5.15+ on storage nodes
- NVMe SSDs strongly recommended
- Hugepages support (2Mi hugepages)
- `nvme-tcp` kernel module

---

## Step 1: Prepare Nodes for V2 Data Engine

```bash
# Install required kernel modules
modprobe nvme-tcp
modprobe uio
modprobe uio_pci_generic

# Make modules persistent
echo "nvme-tcp" >> /etc/modules
echo "uio" >> /etc/modules

# Enable hugepages (required by SPDK)
# Add to /etc/sysctl.d/60-longhorn.conf
echo "vm.nr_hugepages = 1024" >> /etc/sysctl.d/60-longhorn.conf
sysctl -p /etc/sysctl.d/60-longhorn.conf

# Verify hugepages are allocated
cat /proc/meminfo | grep HugePages
```

---

## Step 2: Enable V2 Data Engine in Longhorn

```bash
# Enable V2 data engine globally
kubectl patch setting.longhorn.io v2-data-engine \
  -n longhorn-system \
  --type merge \
  -p '{"value":"true"}'

# Verify the setting
kubectl get setting.longhorn.io v2-data-engine \
  -n longhorn-system \
  -o jsonpath='{.value}'
```

---

## Step 3: Configure Node Hugepages for SPDK

Longhorn's SPDK needs hugepages on each node that will run V2 engine replicas:

```yaml
# Annotate nodes to indicate V2 engine support
kubectl annotate node storage-node-01 \
  node.longhorn.io/hugepages=1024Gi

# Or configure at the OS level with a systemd unit:
cat > /etc/systemd/system/configure-hugepages.service <<EOF
[Unit]
Description=Configure Hugepages for Longhorn V2
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c "echo 1024 > /proc/sys/vm/nr_hugepages"
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
systemctl enable --now configure-hugepages.service
```

---

## Step 4: Create a StorageClass Using V2 Data Engine

```yaml
# storageclass-v2.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-v2
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  # Select the V2 data engine
  dataEngine: "v2"
  diskSelector: "nvme"
  # V2 requires NVMe block devices
  backendStoreDriver: "v2"
```

---

## Step 5: Create a PVC Using V2

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: high-perf-data
  namespace: database
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-v2
  resources:
    requests:
      storage: 100Gi
```

---

## Step 6: Verify V2 Volume Performance

```bash
# Check that the volume is using V2 engine
kubectl get lhvolume <volume-name> -n longhorn-system \
  -o jsonpath='{.spec.dataEngine}'

# Run a quick IOPS benchmark
kubectl run perf-test \
  --image=nixery.dev/shell/fio \
  --restart=Never \
  -- fio --filename=/data/test --rw=randread --bs=4k --iodepth=32 \
     --numjobs=8 --runtime=30 --name=v2-perf-test
```

---

## Best Practices

- V2 Data Engine is still maturing — use V1 for critical production workloads and V2 for performance-sensitive new deployments.
- Dedicate specific nodes to V2 storage using node labels and Longhorn's disk selector.
- Monitor hugepage consumption — SPDK requires consistent hugepage availability to function.
