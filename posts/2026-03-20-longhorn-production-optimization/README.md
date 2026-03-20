# How to Optimize Longhorn Performance for Production

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Performance, Production, Kubernetes, Storage Optimization, Tuning, SUSE Rancher

Description: Learn how to optimize Longhorn for production use, including hardware selection, global settings tuning, network configuration, and monitoring to achieve maximum performance and reliability.

---

Running Longhorn in production requires careful tuning of both hardware and software configurations. This guide provides a comprehensive checklist and configuration recommendations for production deployments.

---

## Hardware Recommendations

| Component | Recommendation |
|---|---|
| Disk | NVMe SSD per node dedicated to Longhorn |
| Network | 10Gbps+ between storage nodes |
| Memory | 16GB+ per node (4GB reserved for Longhorn) |
| CPU | 4+ cores per node |

---

## Step 1: Dedicated Storage Disks

Never use the OS disk for Longhorn data:

```bash
# Format and mount a dedicated disk for Longhorn
mkfs.xfs -f /dev/nvme1n1
mkdir -p /var/lib/longhorn
echo "/dev/nvme1n1 /var/lib/longhorn xfs defaults,noatime,nodiratime 0 0" >> /etc/fstab
mount /var/lib/longhorn

# Verify mount
df -h /var/lib/longhorn
```

---

## Step 2: Key Global Settings

```bash
# Set concurrent rebuild limit per node (1-2 for production)
kubectl patch setting.longhorn.io concurrent-replica-rebuild-per-node-limit \
  -n longhorn-system --type merge -p '{"value":"2"}'

# Configure storage over-provisioning
kubectl patch setting.longhorn.io storage-over-provisioning-percentage \
  -n longhorn-system --type merge -p '{"value":"100"}'

# Set storage minimal available percentage to prevent disk exhaustion
kubectl patch setting.longhorn.io storage-minimal-available-percentage \
  -n longhorn-system --type merge -p '{"value":"15"}'

# Enable auto-cleanup for deleted volume data
kubectl patch setting.longhorn.io orphan-auto-deletion \
  -n longhorn-system --type merge -p '{"value":"true"}'

# Set backup concurrency
kubectl patch setting.longhorn.io backup-concurrent-limit \
  -n longhorn-system --type merge -p '{"value":"5"}'
```

---

## Step 3: Network Optimization

```bash
# Verify Longhorn traffic does not saturate the management network
# Use a dedicated network interface for storage traffic
kubectl patch setting.longhorn.io storage-network \
  -n longhorn-system --type merge \
  -p '{"value":"net-attach-def/storage-net"}'
```

Multus network attachment definition for dedicated storage NIC:

```yaml
# multus-storage-net.yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: storage-net
  namespace: longhorn-system
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "storage-net",
      "type": "ipvlan",
      "master": "eth1",
      "mode": "l3",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.0/24"
      }
    }
```

---

## Step 4: Priority Classes and Resource Limits

```bash
# Create and apply priority class (see priority class guide)
kubectl patch setting.longhorn.io priority-class \
  -n longhorn-system --type merge \
  -p '{"value":"longhorn-critical"}'

# Set guaranteed CPU for instance managers (12% per CPU = 12m)
kubectl patch setting.longhorn.io guaranteed-instance-manager-cpu \
  -n longhorn-system --type merge \
  -p '{"value":"12"}'
```

---

## Step 5: Monitoring and Alerting

Set up the following alerts for production Longhorn:

```yaml
# Essential Longhorn production alerts
- LonghornVolumeFaulted: volume has zero healthy replicas
- LonghornVolumeActualSizeExceedsExpected: volume is unexpectedly large
- LonghornNodeStorageHighUsage: node disk > 80%
- LonghornBackupFailed: backup job failed
- LonghornReplicaRebuildingTooLong: rebuild exceeds 2 hours
```

---

## Production Readiness Checklist

- [ ] Dedicated NVMe disks per node
- [ ] 10Gbps network between storage nodes
- [ ] Priority class configured
- [ ] Backup target configured (S3 or NFS)
- [ ] Recurring backup jobs scheduled
- [ ] Prometheus monitoring and alerts configured
- [ ] `open-iscsi` installed on all nodes
- [ ] Orphaned data cleanup enabled
- [ ] Separate storage network configured (optional but recommended)

---

## Best Practices

- Test the full performance profile with `fio` before going live.
- Establish a baseline of normal IOPS, latency, and throughput to aid future troubleshooting.
- Practice backup restore monthly to ensure your recovery procedure works.
