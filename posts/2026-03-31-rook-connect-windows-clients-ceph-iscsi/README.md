# How to Connect Windows Clients to Ceph via iSCSI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, iSCSI, Windows, Block Storage

Description: Learn how to connect Windows Server clients to Ceph storage via iSCSI, including initiator setup, disk discovery, and CHAP authentication.

---

## Overview

Windows Server includes a built-in iSCSI Initiator that can connect to Ceph iSCSI gateways. This allows Windows clients to use Ceph RBD images as local block devices without any special Ceph drivers.

## Enabling the iSCSI Initiator on Windows

Open the iSCSI Initiator from Server Manager or run:

```powershell
Start-Service MSiSCSI
Set-Service MSiSCSI -StartupType Automatic
```

Or through PowerShell for automated deployment:

```powershell
$service = Get-Service -Name MSiSCSI
if ($service.Status -ne 'Running') {
    Start-Service MSiSCSI
    Set-Service MSiSCSI -StartupType Automatic
    Write-Host "iSCSI Initiator started"
}
```

## Discovering iSCSI Targets

Add the Ceph iSCSI gateway as a discovery portal:

```powershell
New-IscsiTargetPortal -TargetPortalAddress "10.0.1.10"
```

Discover all available targets:

```powershell
Get-IscsiTargetPortal | Update-IscsiTargetPortal
Get-IscsiTarget
```

Expected output:

```text
NodeAddress                                          IsConnected
-----------                                          -----------
iqn.2024-01.com.example:production-storage           False
```

## Connecting to a Target

Connect to the discovered target:

```powershell
Connect-IscsiTarget -NodeAddress "iqn.2024-01.com.example:production-storage" -IsPersistent $true
```

Verify the connection:

```powershell
Get-IscsiSession
Get-IscsiConnection
```

## Configuring CHAP Authentication

If CHAP is enabled on the gateway, provide credentials during connection:

```powershell
Connect-IscsiTarget `
  -NodeAddress "iqn.2024-01.com.example:production-storage" `
  -AuthenticationType ONEWAYCHAP `
  -ChapUsername "initiator1" `
  -ChapSecret "MySecretPassword123" `
  -IsPersistent $true
```

## Initializing the Disk in Windows

After connecting, the new disk appears in Disk Management. Initialize and format it via PowerShell:

```powershell
# Find the new disk
$disk = Get-Disk | Where-Object {$_.OperationalStatus -eq 'Offline'}

# Initialize
Initialize-Disk -Number $disk.Number -PartitionStyle GPT

# Create partition
New-Partition -DiskNumber $disk.Number -UseMaximumSize -AssignDriveLetter

# Format
$partition = Get-Partition -DiskNumber $disk.Number | Where-Object {$_.Type -eq 'Basic'}
Format-Volume -DriveLetter $partition.DriveLetter -FileSystem NTFS -NewFileSystemLabel "CephStorage" -Confirm:$false
```

## Adding a Second Portal for Redundancy

Add the second gateway as a portal for failover:

```powershell
New-IscsiTargetPortal -TargetPortalAddress "10.0.1.11"
Get-IscsiTargetPortal | Update-IscsiTargetPortal
```

Connect again through the second portal (this enables multipath):

```powershell
Connect-IscsiTarget `
  -NodeAddress "iqn.2024-01.com.example:production-storage" `
  -TargetPortalAddress "10.0.1.11" `
  -IsMultipathEnabled $true `
  -IsPersistent $true
```

## Summary

Connecting Windows clients to Ceph via iSCSI requires enabling the built-in iSCSI Initiator, discovering and connecting to the Ceph gateway target, and initializing the resulting disk through Disk Management or PowerShell. CHAP authentication adds security, and connecting through multiple gateway portals enables multipath failover for high availability.
