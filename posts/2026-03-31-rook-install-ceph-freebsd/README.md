# How to Install Ceph on FreeBSD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, FreeBSD, Unix, Installation, Storage, Ports, BSD

Description: Install Ceph on FreeBSD using the Ports collection or pkg binary packages for a BSD-based storage cluster.

---

## Overview

FreeBSD supports Ceph through its Ports collection and pkg binary packages. While Linux is the primary Ceph platform, FreeBSD provides a working Ceph installation for organizations already using FreeBSD infrastructure. Note that Ceph on FreeBSD has some limitations compared to Linux, particularly around kernel client features. This guide covers installation using pkg.

## Prerequisites

- FreeBSD 14.x or newer (recommended)
- At least 3 nodes for a cluster (or 1 for development)
- Additional disk for OSD
- Root access
- Internet access for pkg repository

## Important FreeBSD Limitations

Before proceeding, understand these FreeBSD-specific limitations:

- **No kernel RBD client** - FreeBSD lacks the in-kernel rbd module
- **No CephFS kernel client** - CephFS mounts require FUSE
- **cephadm not supported** - Manual deployment required
- **Container support** - podman/docker support is limited on FreeBSD

These limitations make FreeBSD better suited for Ceph RGW (object storage) servers than full cluster roles.

## Step 1 - Update Ports and Install pkg

```bash
# Update pkg repository
pkg update

# Install required dependencies
pkg install -y python3 leveldb snappy lz4 gperftools rdkafka lua54
```

## Step 2 - Install Ceph via pkg

```bash
# Install Ceph
pkg install -y ceph

# Check installed version
ceph --version
```

Or build from ports for customization:

```bash
# Install portsnap and fetch ports
portsnap fetch extract

# Build Ceph from ports
cd /usr/ports/net/ceph
make config
make install clean
```

## Step 3 - Configure Ceph on FreeBSD

Create the Ceph configuration file:

```ini
# /usr/local/etc/ceph/ceph.conf
[global]
fsid = $(uuidgen)
mon initial members = freebsd-node1
mon host = 192.168.1.10
auth cluster required = cephx
auth service required = cephx
auth client required = cephx
osd pool default size = 3
osd pool default min size = 2

[osd]
osd journal size = 1024
osd data = /var/lib/ceph/osd/ceph-$id
```

## Step 4 - Bootstrap the Monitor

```bash
# Create required directories
mkdir -p /var/lib/ceph/mon/ceph-freebsd-node1
mkdir -p /var/lib/ceph/osd

# Create monitor keyring
ceph-authtool \
  --create-keyring /tmp/ceph.mon.keyring \
  --gen-key \
  -n mon.

# Create admin keyring
ceph-authtool \
  --create-keyring /etc/ceph/ceph.client.admin.keyring \
  --gen-key \
  -n client.admin \
  --cap mon 'allow *' \
  --cap osd 'allow *' \
  --cap mds 'allow *' \
  --cap mgr 'allow *'

# Initialize monitor
ceph-mon --cluster ceph \
  --mkfs \
  -i freebsd-node1 \
  --monmap /tmp/monmap \
  --keyring /tmp/ceph.mon.keyring
```

## Step 5 - Configure OSD on FreeBSD

```bash
# Get OSD UUID and prepare OSD
OSD_UUID=$(uuidgen)
OSD_ID=$(ceph osd create $OSD_UUID)

mkdir -p /var/lib/ceph/osd/ceph-$OSD_ID

# Initialize OSD
ceph-osd --cluster ceph \
  -i $OSD_ID \
  --mkfs \
  --mkkey \
  --osd-uuid $OSD_UUID

# Register the OSD keyring
ceph auth add osd.$OSD_ID osd 'allow *' mon 'allow profile osd' \
  -i /var/lib/ceph/osd/ceph-$OSD_ID/keyring
```

## Step 6 - Enable rc.d Services

```bash
# Enable Ceph services in rc.conf
echo 'ceph_mon_enable="YES"' >> /etc/rc.conf
echo 'ceph_osd_enable="YES"' >> /etc/rc.conf
echo 'ceph_mds_enable="NO"' >> /etc/rc.conf

# Start services
service ceph-mon start
service ceph-osd start
```

## Step 7 - Verify

```bash
ceph -s
ceph osd tree
```

## Summary

Ceph on FreeBSD works primarily for RGW object storage servers and cluster management roles rather than full block/file storage clients due to the absence of kernel RBD and CephFS drivers. Installation uses pkg binary packages or the Ports collection with manual cluster configuration since cephadm is not supported. FreeBSD's ZFS can complement Ceph by providing the underlying storage for OSD directories.
