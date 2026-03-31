# How to Set Up Ceph iSCSI Gateway with LIO

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, iSCSI, LIO, Block Storage

Description: Learn how to set up a Ceph iSCSI gateway using the Linux IO (LIO) target stack to expose Ceph RBD images as iSCSI block devices.

---

## What is the Ceph iSCSI Gateway?

The Ceph iSCSI gateway uses the Linux IO (LIO) target framework and the `ceph-iscsi` package to expose RBD images as iSCSI LUNs. This allows non-native Ceph clients such as Windows servers and VMware ESXi hosts to consume Ceph block storage over standard iSCSI protocols.

## Prerequisites

- Ceph cluster with an RBD pool
- Two dedicated gateway nodes (for HA)
- `ceph-iscsi` and `targetcli` packages

Install the required packages on gateway nodes:

```bash
dnf install -y ceph-iscsi targetcli python3-rtslib python3-configshell
```

## Configuring the RBD Pool

Create a dedicated RBD pool for iSCSI:

```bash
ceph osd pool create iscsi-pool 64 64
rbd pool init iscsi-pool
```

Create an RBD image to expose:

```bash
rbd create --size 100G iscsi-pool/disk1 --image-feature layering
```

## Configuring gwcli

The `gwcli` tool manages the iSCSI gateway configuration:

```bash
gwcli
```

Inside gwcli, configure the gateway:

```
/> cd /iscsi-targets
/iscsi-targets> create iqn.2024-01.com.example:storage
/iscsi-targets/iqn.../> cd gateways
/iscsi-targets/iqn.../gateways> create gateway1 10.0.1.10
/iscsi-targets/iqn.../gateways> create gateway2 10.0.1.11
```

Add the RBD disk:

```
/iscsi-targets/iqn.../> cd disks
/iscsi-targets/iqn.../disks> add rbd iscsi-pool disk1
```

Map the disk to the target LUN:

```
/iscsi-targets/iqn.../hosts> create iqn.1991-05.com.microsoft:initiator1
/iscsi-targets/iqn.../hosts/iqn.../> auth chap=initiator1/Password123
/iscsi-targets/iqn.../hosts/iqn.../luns> add rbd/iscsi-pool/disk1
```

## Configuring ceph-iscsi Service

Save and apply the configuration:

```bash
gwcli export default --file /etc/ceph/iscsi-gateway.cfg
systemctl enable --now rbd-target-api rbd-target-gw
```

Verify the service:

```bash
systemctl status rbd-target-gw
systemctl status rbd-target-api
```

## Verifying the iSCSI Target

Use `targetcli` to verify the target is active:

```bash
targetcli ls
```

Expected output shows an iSCSI target with at least one LUN and portal:

```
o- / .....................................................................
  o- iscsi ................................................................
    o- iqn.2024-01.com.example:storage .................................
      o- tpg1 ...........................................................
        o- acls ..........................................................
        o- luns ..........................................................
          o- lun0 ...................................................../
        o- portals .......................................................
          o- 10.0.1.10:3260 ............................................
```

## Summary

Setting up a Ceph iSCSI gateway with LIO involves creating an RBD pool, installing the `ceph-iscsi` package, configuring targets and LUNs through `gwcli`, and enabling the gateway service. This approach exposes RBD images as standard iSCSI block devices accessible by any iSCSI initiator, bridging Ceph storage with non-native clients.
