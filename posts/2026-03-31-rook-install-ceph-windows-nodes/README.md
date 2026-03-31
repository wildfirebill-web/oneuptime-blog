# How to Install Ceph on Windows Nodes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Windows, Kubernetes, Storage, Installation

Description: Learn how to install and configure Ceph client components on Windows nodes in a Kubernetes cluster to enable Windows workloads to access Ceph storage.

---

## Ceph Windows Support Overview

Ceph provides Windows client support through the `ceph-windows` project, which ports the RADOS, RBD, and CephFS client libraries to Windows. This enables Windows nodes in a Kubernetes cluster to use Ceph-backed persistent volumes.

Key components for Windows:
- `ceph-rbd.exe` - RBD image management
- `ceph-dokan` - CephFS mount via Dokan filesystem driver
- `ceph-iscsi` bridge - for iSCSI-based RBD access

## Prerequisites

Ensure your environment meets these requirements:

```powershell
# Required Windows versions
# Windows Server 2019 or 2022 (build 17763+)
# Windows 10/11 version 1809+

# Check Windows version
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsHardwareAbstractionLayer

# Ensure Hyper-V is available (for WSL2 / container support)
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V
```

## Installing the Ceph Windows Client

Download and install the Ceph Windows package:

```powershell
# Download the latest Ceph Windows installer
$cephVersion = "18.2.4"
Invoke-WebRequest -Uri "https://download.ceph.com/windows/$cephVersion/ceph-$cephVersion-amd64.msi" `
  -OutFile "ceph-windows.msi"

# Install silently
msiexec.exe /i ceph-windows.msi /quiet /norestart

# Verify installation
Get-Command ceph -ErrorAction SilentlyContinue
```

## Configuring Ceph Client on Windows

Create the Ceph configuration directory and files:

```powershell
# Create Ceph config directory
New-Item -ItemType Directory -Force -Path "C:\ProgramData\ceph"

# Create ceph.conf
@"
[global]
fsid = <your-fsid>
mon_host = <mon1-ip>:6789,<mon2-ip>:6789,<mon3-ip>:6789
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
"@ | Set-Content "C:\ProgramData\ceph\ceph.conf"
```

Get the cluster FSID and MON hosts:

```bash
# Run on Linux management node
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mon dump | grep -E "fsid|quorum"
```

## Installing the Ceph Keyring

Create the client keyring on the Windows node:

```powershell
# Create keyring file
@"
[client.windows-node1]
key = AQBxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
"@ | Set-Content "C:\ProgramData\ceph\ceph.client.windows-node1.keyring"
```

Create the client key from your Linux management node:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.windows-node1 \
  mon 'profile rbd' \
  osd 'profile rbd pool=windows-pool'
```

## Testing Connectivity

Verify the Windows node can reach the Ceph cluster:

```powershell
# Test connectivity to MON
ceph -c C:\ProgramData\ceph\ceph.conf status

# List available pools
ceph -c C:\ProgramData\ceph\ceph.conf osd pool ls
```

## Installing the WinFSP and Dokan Drivers

For CephFS filesystem mounting, install the required filesystem drivers:

```powershell
# Install WinFSP (required for CephFS Dokan driver)
winget install WinFsp.WinFsp

# Install Dokan (included with Ceph Windows package)
# Verify Dokan service is running
Get-Service -Name "dokan*"
```

## Summary

Installing Ceph on Windows nodes involves installing the Ceph Windows client package, creating the ceph.conf with cluster connection details, and setting up an authentication keyring scoped to specific pools. For RBD access, the base package is sufficient. For CephFS mounting, the WinFSP and Dokan drivers are additionally required. With these components in place, Windows workloads can access Ceph block and file storage as persistent volumes in your Kubernetes cluster.
