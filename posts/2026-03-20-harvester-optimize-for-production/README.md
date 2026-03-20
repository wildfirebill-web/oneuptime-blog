# How to Optimize Harvester for Production

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Production, Performance, Optimization

Description: A comprehensive guide to optimizing your Harvester cluster for production workloads, covering performance tuning, reliability hardening, and operational best practices.

## Introduction

Running Harvester in production requires more than a basic installation. Production optimization involves hardware tuning (BIOS settings, NUMA topology), network optimization (dedicated NICs, jumbo frames), storage performance (Longhorn tuning, NVMe optimization), and operational practices (monitoring, backups, capacity planning). This guide provides a comprehensive checklist for production-grade Harvester deployments.

## Production Readiness Checklist

```
Hardware:
  ✓ At least 3 nodes for HA
  ✓ IOMMU enabled (for future PCI passthrough)
  ✓ Consistent hardware across nodes
  ✓ Redundant power supplies
  ✓ ECC memory

Networking:
  ✓ Dedicated management NIC
  ✓ Dedicated storage NIC with jumbo frames (MTU 9000)
  ✓ Dedicated VM NIC
  ✓ NIC bonding for redundancy
  ✓ Managed switches with VLAN support

Storage:
  ✓ NVMe SSDs for OS and VM workloads
  ✓ Additional HDDs for bulk storage (tiered storage)
  ✓ Longhorn replica count = 3
  ✓ External backup target configured

Operations:
  ✓ Monitoring enabled (Prometheus + Grafana)
  ✓ Alerting configured
  ✓ Log forwarding configured
  ✓ Backup schedule established
  ✓ DR plan documented and tested
```

## Step 1: BIOS/UEFI Performance Settings

```
Recommended BIOS Settings:
├── Performance Profile: Maximum Performance
├── CPU Power Management: High Performance (disable C-states)
├── NUMA: Enabled
├── Intel Turbo Boost / AMD Precision Boost: Enabled
├── Hyper-Threading: Enabled
├── Memory: XMP/DOCP profile enabled
├── PCIe Link Speed: Gen4 (if supported)
├── SATA Mode: AHCI
└── Fan Control: Performance Mode
```

For CPU power management on the host OS:

```bash
# Set CPU governor to performance mode on all nodes
for NODE in harvester-node-01 harvester-node-02 harvester-node-03; do
    ssh rancher@${NODE} "sudo cpupower frequency-set -g performance"
done

# Verify
ssh rancher@192.168.1.11 "cpupower frequency-info | grep 'current policy'"
```

## Step 2: Kernel Tuning for Production

```bash
# Apply kernel performance tunings on all Harvester nodes
# Create a sysctl configuration file

sudo tee /etc/sysctl.d/99-harvester-production.conf << 'EOF'
# Network performance
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 65536 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_congestion_control = bbr

# Virtual memory
vm.swappiness = 10
vm.dirty_ratio = 60
vm.dirty_background_ratio = 5
vm.min_free_kbytes = 65536

# File descriptors
fs.file-max = 2097152
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 524288

# Hugepages (optional - set based on workload)
# vm.nr_hugepages = 0

# I/O scheduler (for NVMe drives)
# Set via udev rules instead
EOF

# Apply the settings
sudo sysctl --system

# Set I/O scheduler for NVMe to 'none' (already optimal)
# For SSDs, use 'mq-deadline'
for DISK in $(ls /sys/block/ | grep nvme); do
    echo none > /sys/block/${DISK}/queue/scheduler
done

# Make I/O scheduler persistent
cat > /etc/udev/rules.d/60-io-scheduler.rules << 'EOF'
# NVMe drives: use none scheduler (hardware handles queuing)
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/scheduler}="none"
# SSD drives: use mq-deadline
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
EOF
```

## Step 3: Longhorn Storage Optimization

```yaml
# longhorn-production-settings.yaml
# Production-optimized Longhorn settings

---
# Replica count for data durability
apiVersion: longhorn.io/v1beta2
kind: Setting
metadata:
  name: default-replica-count
  namespace: longhorn-system
spec:
  value: "3"

---
# Data locality - keep a replica on the VM's node for reads
apiVersion: longhorn.io/v1beta2
kind: Setting
metadata:
  name: default-data-locality
  namespace: longhorn-system
spec:
  value: "best-effort"

---
# Enable recurring snapshot for data safety
apiVersion: longhorn.io/v1beta2
kind: Setting
metadata:
  name: recurring-successful-jobs-history-limit
  namespace: longhorn-system
spec:
  value: "3"

---
# Concurrent replica rebuilds (don't overload during node failures)
apiVersion: longhorn.io/v1beta2
kind: Setting
metadata:
  name: concurrent-replica-rebuild-per-node-limit
  namespace: longhorn-system
spec:
  value: "3"

---
# Storage overprovisioning - allow 200% of physical capacity
# (because sparse/thin provisioning)
apiVersion: longhorn.io/v1beta2
kind: Setting
metadata:
  name: storage-over-provisioning-percentage
  namespace: longhorn-system
spec:
  value: "200"

---
# Storage minimal available percentage - keep 25% free
apiVersion: longhorn.io/v1beta2
kind: Setting
metadata:
  name: storage-minimal-available-percentage
  namespace: longhorn-system
spec:
  value: "25"
```

```bash
kubectl apply -f longhorn-production-settings.yaml
```

## Step 4: VM Resource Overcommit Policy

```yaml
# overcommit-settings.yaml
# Configure CPU and memory overcommit ratios for VMs

apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: harvester-system
spec:
  configuration:
    developerConfiguration:
      # CPU overcommit ratio: 4x physical CPUs available to VMs
      cpuAllocationRatio: 4
      # Memory overcommit: allow allocating more memory than physical
      memoryOvercommitPercentage: 150
    # Default CPU model for all VMs
    cpuModel: "host-model"
    # Enable offline VM snapshot support
    vmStateStorageClass: "longhorn"
```

## Step 5: Network Performance Tuning

```bash
# Enable jumbo frames on storage and VM NICs
sudo ip link set eth1 mtu 9000  # Storage NIC
sudo ip link set eth2 mtu 9000  # VM NIC

# Verify switch ports also support MTU 9000
ping -M do -s 8972 10.200.0.12  # Test jumbo frames to node 2

# Enable RSS (Receive Side Scaling) for multi-queue networking
sudo ethtool -L eth0 combined 8  # Set 8 queues (match CPU count)
sudo ethtool -L eth1 combined 8

# Enable TCP offloading for performance
sudo ethtool -K eth0 tso on gso on gro on
sudo ethtool -K eth1 tso on gso on gro on
```

## Step 6: Configure Resource Reservations

Reserve resources for the host OS and Kubernetes system pods:

```yaml
# kubelet-config.yaml
# Reserve resources for system processes

apiVersion: v1
kind: ConfigMap
metadata:
  name: kubelet-config
  namespace: kube-system
data:
  kubelet: |
    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    # Reserve resources for system processes (not allocatable to VMs)
    systemReserved:
      cpu: "1000m"
      memory: "2Gi"
      ephemeral-storage: "10Gi"
    # Reserve resources for Kubernetes system pods
    kubeReserved:
      cpu: "1000m"
      memory: "2Gi"
      ephemeral-storage: "10Gi"
    # Evict VMs before the node runs out of resources
    evictionHard:
      memory.available: "5%"
      nodefs.available: "10%"
      nodefs.inodesFree: "5%"
    # CPU management for better VM performance
    cpuManagerPolicy: static
    topologyManagerPolicy: best-effort
```

## Step 7: HA Configuration Validation

```bash
# Test that the cluster survives a single node failure
# Identify the current VIP holder
for NODE in 192.168.1.11 192.168.1.12 192.168.1.13; do
    ssh rancher@${NODE} "ip addr show | grep 192.168.1.100" 2>/dev/null && \
        echo "${NODE} holds the VIP" || true
done

# Simulate node failure
ssh rancher@192.168.1.11 "sudo systemctl stop rke2-server"

# Verify cluster remains operational
sleep 30
kubectl get nodes  # Should show node-01 as NotReady, others Ready
kubectl get vmi -A  # VMs should still be running

# Restore the node
ssh rancher@192.168.1.11 "sudo systemctl start rke2-server"
kubectl wait node/harvester-node-01 --for=condition=Ready --timeout=300s
```

## Step 8: Production Monitoring Thresholds

```promql
# Alert thresholds for production Harvester

# CPU Overload (> 80% for 10 minutes)
avg by (node) (1 - rate(node_cpu_seconds_total{mode="idle"}[5m])) > 0.80

# Memory Pressure (> 90%)
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.90

# Storage Low (< 20% free)
1 - (node_filesystem_free_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) > 0.80

# Longhorn Degraded Volumes
longhorn_volume_robustness{robustness="degraded"} > 0

# etcd Disk Latency (> 10ms is warning, > 25ms is critical)
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) > 0.01
```

## Conclusion

Production-optimizing Harvester is an iterative process that spans hardware selection, OS tuning, storage configuration, and operational practices. The most impactful optimizations are typically: dedicated NICs with traffic separation, NVMe storage with appropriate Longhorn settings, and proper resource reservations to prevent noisy-neighbor problems. Establish your monitoring and alerting early, run thorough HA testing before go-live, and regularly review capacity utilization to stay ahead of resource exhaustion. A well-tuned Harvester cluster delivers excellent VM density and performance while maintaining the operational simplicity that makes HCI attractive.
