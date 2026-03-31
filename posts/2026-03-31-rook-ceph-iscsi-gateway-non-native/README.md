# How to Set Up Ceph iSCSI Gateway for Non-Native Clients

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, iSCSI, Gateway, Storage, Block Storage, Non-Native

Description: Deploy the Ceph iSCSI gateway in Rook to expose RBD images over the iSCSI protocol, enabling non-native clients like Windows servers and legacy systems to access Ceph storage.

---

## Overview

Not all systems support the native Ceph RADOS protocol. The Ceph iSCSI gateway (ceph-iscsi) exposes RBD images as iSCSI targets, enabling Windows servers, VMware ESXi, and other non-native clients to use Ceph block storage through the universal iSCSI protocol.

## Prerequisites

- Rook-Ceph cluster v1.10+
- At least two dedicated gateway nodes with two network interfaces
- iSCSI initiator on client systems

## Step 1 - Enable the iSCSI Gateway in Rook

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: iscsi-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
```

```yaml
apiVersion: ceph.rook.io/v1
kind: CephIscsiGateway
metadata:
  name: rook-iscsi
  namespace: rook-ceph
spec:
  image: quay.io/ceph/ceph:v18.2.0
  gatewayNodes:
    - name: gateway-node1
    - name: gateway-node2
  livenessProbe:
    disabled: false
  readinessProbe:
    disabled: false
```

```bash
kubectl apply -f iscsi-gateway.yaml
kubectl -n rook-ceph get pod -l app=rook-ceph-iscsi
```

## Step 2 - Create an RBD Image for iSCSI

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  rbd create iscsi-pool/iscsi-disk-01 \
  --size 100G \
  --image-feature layering
```

## Step 3 - Configure the iSCSI Target

Access the ceph-iscsi API:

```bash
GATEWAY_IP=$(kubectl -n rook-ceph get svc rook-ceph-iscsi-service -o jsonpath='{.spec.clusterIP}')

# Using gwcli to configure targets
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

gwcli <<EOF
/iscsi-targets create iqn.2016-06.io.rook:target1
/iscsi-targets/iqn.2016-06.io.rook:target1/gateways create gateway1 ${GATEWAY_IP} skipchecks=true
/iscsi-targets/iqn.2016-06.io.rook:target1/hosts create iqn.2024-01.com.example:initiator1
/iscsi-targets/iqn.2016-06.io.rook:target1/hosts/iqn.2024-01.com.example:initiator1 chap_creds username=backupuser password=secretpassword123
/iscsi-targets/iqn.2016-06.io.rook:target1/disks add pool=iscsi-pool image=iscsi-disk-01
/iscsi-targets/iqn.2016-06.io.rook:target1/disks/iscsi-pool.iscsi-disk-01 add_acl initiator iqn.2024-01.com.example:initiator1
EOF
```

## Step 4 - Connect an iSCSI Initiator (Linux)

On the client:

```bash
apt-get install -y open-iscsi

# Discover targets
iscsiadm -m discovery -t st -p GATEWAY_IP:3260

# Connect to target
iscsiadm -m node \
  -T iqn.2016-06.io.rook:target1 \
  -p GATEWAY_IP:3260 \
  --login

# Verify the block device appeared
lsblk | grep sd
```

## Step 5 - Connect from Windows

In Windows Server:
1. Open iSCSI Initiator (Server Manager -> Tools -> iSCSI Initiator)
2. Target tab -> Enter gateway IP -> Quick Connect
3. Discovery tab -> Add gateway IP for discovery
4. After connecting, use Disk Management to format and mount the new disk

## Step 6 - Configure Multipathing

For redundancy through two gateways:

```bash
# Linux multipath
apt-get install -y multipath-tools
systemctl enable --now multipathd

# Add both gateway IPs
iscsiadm -m discovery -t st -p GATEWAY_IP1:3260
iscsiadm -m discovery -t st -p GATEWAY_IP2:3260
iscsiadm -m node --loginall all

# Verify multipath
multipath -ll
```

## Summary

The Ceph iSCSI gateway bridges Ceph's native block storage with universal iSCSI protocol support, enabling any iSCSI-capable client to use Ceph RBD images as block devices. Using multiple gateway nodes with multipathing provides both redundancy and load balancing. The gateway handles the translation between iSCSI commands and Ceph RADOS operations transparently.
