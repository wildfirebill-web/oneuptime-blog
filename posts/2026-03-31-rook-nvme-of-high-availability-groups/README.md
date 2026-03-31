# How to Configure NVMe-oF High Availability Groups in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, NVMe-oF, High Availability, Storage

Description: Configure NVMe-oF high availability groups in Ceph to ensure gateway failover and continuous storage access when individual gateway nodes fail.

---

## Overview

Ceph NVMe-oF supports high availability (HA) through gateway groups. Multiple gateway instances form an HA group where namespaces are load-balanced and failed gateways trigger automatic failover to surviving group members.

## Understanding NVMe-oF HA Architecture

In an HA configuration:
- Multiple gateway pods form a gateway group
- Each namespace is owned by one gateway but accessible via all gateways
- On failure, the surviving gateway takes ownership
- Initiators use ANA (Asymmetric Namespace Access) to prefer the owning gateway

## Deploy a Multi-Gateway Configuration

```yaml
apiVersion: ceph.rook.io/v1
kind: CephNVMeoFGateway
metadata:
  name: nvmeof-ha-gw
  namespace: rook-ceph
spec:
  server:
    active: 2           # 2 active gateways
  pool:
    name: nvmeof-pool
```

Verify two gateway pods are running:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-nvmeof
# Should show 2 pods in Running state
```

## Configure the HA Group

```bash
# List current gateways
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph nvmeof gateway list

# Set redundancy level for the HA group
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph nvmeof gateway set_redundancy_count \
  --gateway-name nvmeof-ha-gw \
  --redundancy-count 2
```

## Create Subsystem with HA Listeners

Add listeners on both gateways for the same subsystem:

```bash
NQN="nqn.2024-01.io.ceph:ha-subsystem"

# Create subsystem
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph nvmeof subsystem create --nqn $NQN

# Add listener on gateway 1
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph nvmeof gateway add_listener \
  --nqn $NQN --host-name nvmeof-ha-gw-0 \
  --traddr 10.0.1.10 --trsvcid 4420 --trtype TCP

# Add listener on gateway 2
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph nvmeof gateway add_listener \
  --nqn $NQN --host-name nvmeof-ha-gw-1 \
  --traddr 10.0.1.11 --trsvcid 4420 --trtype TCP
```

## Verify ANA Groups

```bash
# Check ANA group assignments after namespace creation
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph nvmeof namespace list --nqn $NQN

# Each namespace should show optimized/non-optimized paths
```

## Test Failover

Simulate a gateway failure and verify continuity:

```bash
# On the initiator node
nvme list
nvme show-ana /dev/nvme0

# Kill one gateway pod
kubectl -n rook-ceph delete pod rook-ceph-nvmeof-ha-gw-0-xxxxx

# On initiator - verify path switches to second gateway
nvme show-ana /dev/nvme0  # Should show path changed
```

## Summary

NVMe-oF HA groups in Ceph provide automatic failover by deploying multiple gateway instances that share subsystem listeners. ANA (Asymmetric Namespace Access) allows initiators to prefer the optimal path while automatically failing over to surviving gateways. The CephNVMeoFGateway resource `active` count controls the number of HA group members.
