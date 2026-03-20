# How to Optimize Harvester for Production - Optimization

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Production, Optimization, HCI, Performance, Kubernetes, SUSE Rancher

Description: Learn how to optimize Harvester HCI for production deployments including hardware configuration, Longhorn tuning, network optimization, and reliability settings for maximum VM density and performance.

---

Running Harvester in production requires hardware selection, operating system tuning, and Longhorn and Kubernetes configuration optimization. This guide provides a production-readiness checklist and configuration recommendations.

---

## Hardware Sizing Recommendations

| Component | Minimum | Recommended |
|---|---|---|
| CPU | 16 cores | 32+ cores (AMD EPYC or Intel Xeon) |
| RAM | 32GB | 128GB+ (64GB per 10 VMs) |
| Storage | SATA SSD | NVMe SSD (dedicated for Longhorn) |
| Network | 1Gbps | 25Gbps (management) + 25Gbps (storage) |
| Nodes | 3 | 5+ for HA |

---

## Step 1: OS-Level Tuning

Apply these settings on each Harvester node:

```bash
# Increase inotify limits for high VM density

cat >> /etc/sysctl.d/99-harvester.conf <<EOF
# Increase network buffers
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.netdev_max_backlog = 5000

# VM memory management
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
vm.swappiness = 10

# File descriptor limits
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 1024
EOF
sysctl -p /etc/sysctl.d/99-harvester.conf
```

---

## Step 2: Longhorn Tuning for Production

```bash
# Set storage minimal available to 15% (Longhorn stops scheduling at this threshold)
kubectl patch setting.longhorn.io storage-minimal-available-percentage \
  -n longhorn-system --type merge -p '{"value":"15"}'

# Limit concurrent rebuilds to protect production performance
kubectl patch setting.longhorn.io concurrent-replica-rebuild-per-node-limit \
  -n longhorn-system --type merge -p '{"value":"1"}'

# Set priority class for Longhorn system pods
kubectl patch setting.longhorn.io priority-class \
  -n longhorn-system --type merge -p '{"value":"longhorn-critical"}'
```

---

## Step 3: Network Separation

Configure separate VLANs for different traffic types:

```yaml
# Example VLAN configuration for Harvester
# In Harvester UI: Hosts > [Node] > Network Config

# Recommended VLAN assignment:
# VLAN 10: Management (Harvester API, Rancher agent)
# VLAN 20: Storage (Longhorn replication traffic)
# VLAN 30: VM traffic (production workloads)
# VLAN 40: VM traffic (development/testing)
```

---

## Step 4: Configure Resource Overcommit Ratios

```bash
# Set CPU overcommit ratio (2.0 = 2x CPU overcommit)
kubectl patch kubevirt kubevirt -n harvester-system --type merge -p '{
  "spec": {
    "configuration": {
      "developerConfiguration": {
        "cpuAllocationRatio": 10
      }
    }
  }
}'
```

---

## Step 5: Enable Live Migration Performance

```bash
# Configure live migration bandwidth limit to protect storage network
kubectl patch kubevirt kubevirt -n harvester-system --type merge -p '{
  "spec": {
    "configuration": {
      "migrations": {
        "bandwidthPerMigration": "512Mi",
        "completionTimeoutPerGiB": 800,
        "progressTimeout": 150
      }
    }
  }
}'
```

---

## Production Readiness Checklist

- [ ] Minimum 3 nodes (5+ recommended for HA)
- [ ] Dedicated NVMe SSDs for Longhorn
- [ ] 25Gbps+ network interfaces
- [ ] Separate network interfaces for storage and management
- [ ] BIOS SR-IOV and VT-d enabled
- [ ] OS sysctl tuning applied
- [ ] Longhorn priority classes configured
- [ ] Prometheus monitoring and alerting enabled
- [ ] Backup target configured (S3 or NFS)
- [ ] Rancher integration configured
- [ ] Tested live migration successfully

---

## Best Practices

- Never run a 2-node Harvester cluster in production - etcd requires an odd quorum.
- Monitor VM density per node and alert when it exceeds planned capacity.
- Schedule regular maintenance windows for OS updates using Harvester's built-in drain workflow.
