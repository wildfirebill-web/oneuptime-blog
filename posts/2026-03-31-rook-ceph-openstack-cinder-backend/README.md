# How to Set Up Ceph as OpenStack Cinder Backend

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, OpenStack, Cinder, Block Storage, RBD, Cloud

Description: Configure Ceph RBD as the backend for OpenStack Cinder to provide scalable, resilient block storage volumes for OpenStack instances.

---

OpenStack Cinder manages block storage volumes for VMs. Using Ceph RBD as the Cinder backend gives you thin-provisioned volumes, snapshots, and live migration support without a SAN. This guide walks through the complete integration.

## Prerequisites

- A running Ceph cluster (Reef or newer recommended)
- OpenStack (Zed or newer) with Cinder installed
- Network connectivity between Cinder nodes and Ceph monitors

## Step 1: Create a Dedicated Ceph Pool

Create a pool specifically for Cinder volumes and initialize it for RBD:

```bash
# Create the volumes pool
ceph osd pool create volumes 128

# Initialize the pool for RBD use
rbd pool init volumes

# Set replication
ceph osd pool set volumes size 3
ceph osd pool set volumes min_size 2
```

## Step 2: Create a Ceph User for Cinder

```bash
# Create the cinder user with appropriate caps
ceph auth get-or-create client.cinder \
  mon 'profile rbd' \
  osd 'profile rbd pool=volumes, profile rbd pool=images, profile rbd-read-only pool=backups' \
  mgr 'profile rbd pool=volumes' \
  -o /etc/ceph/ceph.client.cinder.keyring
```

## Step 3: Distribute Ceph Configuration to Cinder Nodes

Copy the Ceph config and keyring to every node running cinder-volume:

```bash
# On the Ceph admin node
scp /etc/ceph/ceph.conf cinder-node:/etc/ceph/ceph.conf
scp /etc/ceph/ceph.client.cinder.keyring cinder-node:/etc/ceph/ceph.client.cinder.keyring

# Set permissions on the Cinder node
ssh cinder-node "chmod 640 /etc/ceph/ceph.client.cinder.keyring && \
  chown root:cinder /etc/ceph/ceph.client.cinder.keyring"
```

## Step 4: Configure Cinder to Use Ceph

Edit `/etc/cinder/cinder.conf` on the Cinder node:

```ini
[DEFAULT]
enabled_backends = ceph

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
rbd_user = cinder
rbd_secret_uuid = <libvirt-secret-uuid>
```

## Step 5: Configure libvirt Secret for Nova

Nova needs a libvirt secret to authenticate to Ceph when attaching volumes:

```bash
# Generate a UUID for the secret
SECRET_UUID=$(uuidgen)

# Create the secret XML
cat > /tmp/secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>${SECRET_UUID}</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF

# Define and set the secret value
virsh secret-define --file /tmp/secret.xml
CINDER_KEY=$(ceph auth get-key client.cinder)
virsh secret-set-value --secret ${SECRET_UUID} --base64 ${CINDER_KEY}
```

## Step 6: Restart Cinder and Verify

```bash
# Restart cinder-volume service
systemctl restart openstack-cinder-volume

# Check that the backend is registered
openstack volume service list

# Create a test volume
openstack volume create --size 10 test-ceph-vol

# Verify the RBD image was created
rbd -p volumes ls
```

## Summary

Setting up Ceph as an OpenStack Cinder backend involves creating a dedicated RBD pool, provisioning a Ceph auth user, distributing credentials to Cinder nodes, and configuring the RBD volume driver. Once configured, Cinder volumes are backed by Ceph RBD images, enabling features like copy-on-write cloning, snapshots, and seamless live migration between Nova compute nodes.
