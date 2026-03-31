# How to Clone VMs Using Ceph RBD Copy-on-Write

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Clone, VM, Copy-on-Write, Virtualization

Description: Use Ceph RBD's copy-on-write clone feature to rapidly provision new VM instances from golden images, consuming no extra disk space until changes are written.

---

## Overview

Ceph RBD clones allow you to create new RBD images that initially share all data with a parent snapshot using copy-on-write. New VMs can be provisioned from a "golden" template image in seconds, with each clone only storing its own changes.

## How RBD COW Clones Work

1. Create a parent image (base VM disk)
2. Take a snapshot of the parent and protect it
3. Clone the snapshot - each clone starts sharing parent data
4. As the VM writes new data, only changed blocks are stored in the clone
5. Clone size grows only with writes - initial size is near zero

## Step 1 - Create a Golden VM Image

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- bash

# Create a base image and install OS (via QEMU)
rbd create vms/ubuntu-22.04-golden --size 30G --image-feature layering

# The image gets populated with an OS installation
# Then create the final snapshot
rbd snap create vms/ubuntu-22.04-golden@v1.0
rbd snap protect vms/ubuntu-22.04-golden@v1.0
```

## Step 2 - Clone the Golden Image for a New VM

```bash
# Clone - nearly instant, no data copied
rbd clone vms/ubuntu-22.04-golden@v1.0 vms/webapp-vm-01
rbd clone vms/ubuntu-22.04-golden@v1.0 vms/webapp-vm-02
rbd clone vms/ubuntu-22.04-golden@v1.0 vms/webapp-vm-03

# All three clones created immediately
rbd ls vms | grep webapp
```

## Step 3 - View Clone Relationships

```bash
# Check parent of a clone
rbd info vms/webapp-vm-01 | grep parent

# List all children of the golden snapshot
rbd children vms/ubuntu-22.04-golden@v1.0
```

## Step 4 - Start VMs from Clones

```bash
# Use the clone as a VM disk directly
virsh define webapp-vm-01.xml  # XML referencing vms/webapp-vm-01
virsh start webapp-vm-01
```

## Step 5 - Flatten a Clone (Break Parent Dependency)

When you want a clone to become independent:

```bash
# Flatten copies all parent data to the clone
# Use before upgrading the golden image
rbd flatten vms/webapp-vm-01

# Now independent - safe to modify golden image
rbd info vms/webapp-vm-01 | grep parent
# Should show no parent
```

## Step 6 - Update the Golden Image

```bash
# Unprotect old snapshot
rbd snap unprotect vms/ubuntu-22.04-golden@v1.0

# Only works when no more dependent clones exist
# First flatten all clones, then unprotect

# Create new golden snapshot
rbd snap create vms/ubuntu-22.04-golden@v1.1
rbd snap protect vms/ubuntu-22.04-golden@v1.1
```

## Step 7 - Automate VM Provisioning

```bash
#!/bin/bash
GOLDEN=vms/ubuntu-22.04-golden
GOLDEN_SNAP="${GOLDEN}@v1.0"
VM_NAME=$1

if [ -z "$VM_NAME" ]; then
  echo "Usage: $0 vm-name"
  exit 1
fi

echo "Cloning $GOLDEN_SNAP -> vms/${VM_NAME}"
rbd clone $GOLDEN_SNAP vms/${VM_NAME}

echo "New VM disk ready: vms/${VM_NAME}"
echo "Disk info:"
rbd info vms/${VM_NAME}
```

## Storage Efficiency Example

```bash
# Check disk usage of clones vs base image
rbd du vms/ubuntu-22.04-golden
rbd du vms/webapp-vm-01
```

Example output after one hour of use:

```
NAME                  PROVISIONED  USED
ubuntu-22.04-golden   30 GiB       8 GiB
webapp-vm-01          30 GiB       512 MiB
webapp-vm-02          30 GiB       384 MiB
```

Each clone only stores its own changes.

## Summary

Ceph RBD COW clones transform VM provisioning from a disk-copy operation taking minutes into an instant snapshot operation. A pool of 100 VMs from one golden image can be provisioned in under a minute while consuming minimal additional storage. The clone relationship to the parent image enables efficient golden-image-based update workflows where you update the parent and re-provision clones from the new snapshot.
