# How to Configure NVMe over TCP for Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, NVMe, Storage, Networking, Performance

Description: Set up NVMe over TCP (NVMe/TCP) with Ceph to expose RBD block devices as NVMe namespaces over standard TCP networks for high-performance storage access.

---

## What is NVMe over TCP with Ceph?

Ceph supports exposing RBD images as NVMe namespaces using the NVMe over TCP (NVMe/TCP) protocol via the `nvmeof` gateway. This allows NVMe initiators to access Ceph block storage over standard Ethernet without Fibre Channel or RDMA infrastructure, achieving near-native NVMe performance.

## Prerequisites

- Ceph 18.x (Reef) or later with nvmeof gateway support
- Linux kernel 5.0+ on initiator nodes (for NVMe/TCP driver)
- Rook v1.13+ with NVMe-oF gateway CRD support

## Deploying the NVMe-oF Gateway

Enable the NVMe-oF gateway in your Rook CephCluster:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephNVMEoFGateway
metadata:
  name: nvmeof-gateway
  namespace: rook-ceph
spec:
  cephCluster:
    name: rook-ceph
  pool: nvmeof-pool
  gatewaySpec:
    serviceType: NodePort
    port: 4420
    instances: 2
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "2000m"
```

Create a dedicated pool for NVMe-oF:

```bash
ceph osd pool create nvmeof-pool 128 128
ceph osd pool application enable nvmeof-pool rbd
rbd pool init nvmeof-pool
```

## Creating NVMe Subsystems and Namespaces

Use the `nvmeof` CLI to create a subsystem:

```bash
# Access the nvmeof gateway pod
kubectl exec -it -n rook-ceph deploy/rook-ceph-nvmeof-gateway -- bash

# Create a subsystem
nvmeof-cli subsystem create --subsystem-nqn nqn.2024-01.io.ceph:nvmeof-subsystem-1

# Create an RBD image
rbd create nvmeof-pool/nvmeof-image-1 --size 100G

# Add the image as a namespace
nvmeof-cli namespace add \
  --subsystem-nqn nqn.2024-01.io.ceph:nvmeof-subsystem-1 \
  --rbd-pool nvmeof-pool \
  --rbd-image nvmeof-image-1
```

List configured subsystems:

```bash
nvmeof-cli subsystem list
nvmeof-cli namespace list --subsystem-nqn nqn.2024-01.io.ceph:nvmeof-subsystem-1
```

## Connecting from an Initiator

Load the NVMe/TCP kernel module on the initiator:

```bash
modprobe nvme-tcp
```

Discover available subsystems:

```bash
nvme discover -t tcp -a <gateway-ip> -s 4420 -o normal
```

Connect to a subsystem:

```bash
nvme connect \
  -t tcp \
  -a <gateway-ip> \
  -s 4420 \
  -n nqn.2024-01.io.ceph:nvmeof-subsystem-1
```

Verify the connection:

```bash
nvme list
nvme list-subsys
```

The device appears as `/dev/nvme0n1` or similar and can be formatted and used:

```bash
mkfs.ext4 /dev/nvme0n1
mount /dev/nvme0n1 /mnt/nvmeof
```

## Performance Tuning for NVMe/TCP

Optimize TCP settings for low-latency NVMe traffic:

```bash
# Increase TCP socket buffers
sysctl -w net.core.rmem_max=67108864
sysctl -w net.core.wmem_max=67108864
sysctl -w net.ipv4.tcp_rmem="4096 87380 67108864"
sysctl -w net.ipv4.tcp_wmem="4096 87380 67108864"

# Disable TCP Nagle for lower latency
echo 1 > /sys/class/net/eth0/queues/tx-0/xps_cpus
```

Make persistent in `/etc/sysctl.d/99-nvmeof.conf`:

```ini
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 87380 67108864
```

## Summary

Ceph's NVMe-oF gateway allows exposing RBD images as NVMe namespaces over standard TCP networks, providing high-performance block storage access without specialized hardware. Rook's CephNVMEoFGateway CRD simplifies deployment in Kubernetes environments. Tuning TCP socket buffers on initiator nodes ensures the TCP transport overhead does not limit throughput.
