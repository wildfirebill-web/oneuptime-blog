# How to Use RBD on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Window, Block Storage

Description: Learn how to access Ceph RBD block storage from Windows clients using the Windows RBD driver and iSCSI gateway in Rook-Ceph environments.

---

## RBD on Windows Overview

Windows clients can access Ceph RBD storage through two methods:

1. **Windows RBD Driver (WNBD)** - Native RBD driver for Windows, provides direct protocol support
2. **iSCSI Gateway** - Exposes RBD images as iSCSI LUNs, works with any Windows version

The WNBD approach is newer and offers better integration, while iSCSI works universally but requires a gateway.

## Method 1 - Using the Windows RBD Driver (WNBD)

### Step 1 - Install the Ceph Windows Package

Download and install the Ceph Windows package on the Windows host:

```powershell
# Download from the Ceph Windows releases
Invoke-WebRequest -Uri "https://cloudbase.it/ceph-for-windows/" -OutFile ceph-windows-installer.msi
Start-Process msiexec.exe -Wait -ArgumentList '/I ceph-windows-installer.msi /quiet'
```

### Step 2 - Configure Ceph Credentials

Create `C:\ProgramData\ceph\ceph.conf`:

```text
[global]
mon_host = 192.168.1.10:6789,192.168.1.11:6789,192.168.1.12:6789
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
log_to_stderr = true
```

Create `C:\ProgramData\ceph\ceph.client.admin.keyring`:

```text
[client.admin]
    key = <your-ceph-admin-key>
```

Extract the key from Rook:

```bash
kubectl get secret rook-ceph-admin-keyring -n rook-ceph \
  -o jsonpath='{.data.keyring}' | base64 -d
```

### Step 3 - Map an RBD Image

Open a Command Prompt as Administrator and map the RBD image:

```powershell
rbd-wnbd map replicapool/myimage
```

Verify the mapping:

```powershell
rbd-wnbd show
```

The image appears as a new disk in Disk Management.

### Step 4 - Format and Use the Disk

```powershell
Get-Disk | Where-Object PartitionStyle -eq "RAW" | Initialize-Disk -PartitionStyle GPT
New-Partition -DiskNumber 1 -UseMaximumSize -AssignDriveLetter | Format-Volume -FileSystem NTFS -NewFileSystemLabel "CephRBD"
```

## Method 2 - Via iSCSI Gateway

For older Windows versions or environments without the WNBD driver:

### Step 1 - Set Up an iSCSI Gateway

Deploy the Ceph iSCSI gateway and expose the RBD image as an iSCSI target (refer to the LIO iSCSI gateway guide).

### Step 2 - Connect via Windows iSCSI Initiator

```powershell
# Discover targets
iscsicli QAddTargetPortal 192.168.1.20

# Connect to target
iscsicli QLoginTarget iqn.2026-03.com.example:ceph-lun-01
```

Or use the GUI: Server Manager > iSCSI Initiator > Discovery tab.

## Step 3 - Persistent Mapping

Configure the RBD image to map automatically at Windows startup:

```powershell
rbd-wnbd map replicapool/myimage --persistent
rbd-wnbd service-start
```

## Step 4 - Verify from Rook Side

Confirm the Windows client is connected:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd status replicapool/myimage
```

```text
Watchers:
        watcher=192.168.1.100:0/1234 client.56789 cookie=18446744073709551615
```

## Summary

Windows clients can access Ceph RBD storage using either the native WNBD driver or via an iSCSI gateway. The WNBD approach provides direct RBD protocol access with automatic drive mapping, while iSCSI works with any Windows version. Configure Ceph credentials on the Windows host from Rook-managed secrets, and use `rbd-wnbd map --persistent` to ensure volumes remount after reboots.
