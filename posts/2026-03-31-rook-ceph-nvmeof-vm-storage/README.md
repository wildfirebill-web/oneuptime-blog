# How to Configure Ceph NVMe-oF for High-Performance VM Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, NVMe-oF, VM, Storage, Performance, Block Storage

Description: Deploy Ceph NVMe-oF gateways in Rook to expose RBD images over NVMe over Fabrics, providing low-latency high-throughput block storage for demanding VM workloads.

---

## Overview

NVMe over Fabrics (NVMe-oF) extends the NVMe protocol over TCP or RDMA networks, providing significantly lower latency than iSCSI. Ceph supports NVMe-oF through the SPDK-based NVMe-oF gateway, making it ideal for latency-sensitive VM workloads like databases.

## Prerequisites

- Rook-Ceph cluster v1.12+
- Gateway nodes with 25GbE+ networking for best performance
- NVMe-oF initiator support on client nodes (kernel 5.0+ for TCP)

## Step 1 - Create a High-Performance Pool

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: nvmeof-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
  parameters:
    compression_mode: none
    fast_read: "1"
```

```bash
kubectl apply -f nvmeof-pool.yaml
```

## Step 2 - Deploy the NVMe-oF Gateway

```yaml
apiVersion: ceph.rook.io/v1
kind: CephNVMEofGateway
metadata:
  name: nvmeof-gw
  namespace: rook-ceph
spec:
  gatewayConfig:
    pool: nvmeof-pool
    serviceId: gateway-service
    gatewayServerCert: ""
  ips:
  - ip: 10.0.0.100
  - ip: 10.0.0.101
```

```bash
kubectl apply -f nvmeof-gateway.yaml
kubectl -n rook-ceph get pod -l app=ceph-nvmeof
```

## Step 3 - Create an NVMe-oF Subsystem

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- bash

# Create a subsystem
ceph nvme-gw create nvmeof-pool gateway-service

# Create an RBD image for NVMe-oF
rbd create nvmeof-pool/nvme-disk-01 --size 200G --image-feature layering

# Expose the image via NVMe-oF
ceph nvme-gw subsystem create \
  nvmeof-pool gateway-service \
  nqn.2024-01.io.ceph:nvme1
```

## Step 4 - Configure via the NVMe-oF CLI

```bash
kubectl -n rook-ceph exec -it $(kubectl get pod -n rook-ceph -l app=ceph-nvmeof -o name | head -1) -- bash

python3 -c "
import grpc
import nvmeof_gateway_pb2 as pb2
import nvmeof_gateway_pb2_grpc as pb2_grpc

channel = grpc.insecure_channel('localhost:5500')
stub = pb2_grpc.GatewayStub(channel)

# Create subsystem
stub.create_subsystem(pb2.create_subsystem_req(
    subsystem_nqn='nqn.2024-01.io.ceph:nvme1',
    serial_number='CEPH0000001',
    max_namespaces=256
))

# Add namespace (maps RBD image)
stub.add_namespace(pb2.add_namespace_req(
    subsystem_nqn='nqn.2024-01.io.ceph:nvme1',
    rbd_image_name='nvme-disk-01',
    rbd_pool_name='nvmeof-pool',
    nsid=1
))
"
```

## Step 5 - Connect from a Linux Client

```bash
# Install NVMe-oF tools
apt-get install -y nvme-cli

# Load the NVMe-oF TCP kernel module
modprobe nvme-tcp

# Discover subsystems
nvme discover -t tcp -a 10.0.0.100 -s 4420

# Connect to the subsystem
nvme connect \
  -t tcp \
  -n nqn.2024-01.io.ceph:nvme1 \
  -a 10.0.0.100 \
  -s 4420

# List connected NVMe devices
nvme list
```

## Step 6 - Performance Tuning

```bash
# Increase queue depth on the NVMe device
echo 64 > /sys/block/nvme0n1/queue/nr_requests

# Set I/O scheduler to none for NVMe
echo none > /sys/block/nvme0n1/queue/scheduler

# Check IOPS
fio --filename=/dev/nvme0n1 \
  --rw=randread \
  --bs=4k \
  --iodepth=64 \
  --numjobs=4 \
  --runtime=30 \
  --name=nvme-test
```

## Summary

Ceph NVMe-oF gateways provide a significant performance improvement over iSCSI for latency-sensitive workloads by using the NVMe command set over TCP or RDMA networks. For VM storage, NVMe-oF delivers near-native NVMe latency characteristics while maintaining all Ceph benefits like replication, snapshots, and elastic scaling. TCP transport makes it deployable without specialized RDMA hardware.
