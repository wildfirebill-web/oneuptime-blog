# How to Add Windows Worker Nodes to Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Windows, Kubernetes, Worker Nodes, Hybrid Cluster

Description: Add Windows Server worker nodes to Rancher-managed Kubernetes clusters to run Windows containers alongside Linux workloads in a hybrid cluster configuration.

## Introduction

Rancher supports hybrid Kubernetes clusters with both Linux and Windows worker nodes. Windows nodes can run Windows-specific workloads like .NET Framework applications, IIS, and SQL Server, while Linux nodes handle Linux-native workloads. This guide covers adding Windows Server nodes to a Rancher-managed cluster.

## Prerequisites

- RKE2 cluster with Linux control plane nodes
- Windows Server 2019 or 2022 (Windows 10/11 for dev)
- Network plugin supporting Windows (Flannel or Calico)
- At minimum Windows version: Windows Server 2019 (1809+)
- Rancher v2.6+ for RKE2 Windows support

## Step 1: Prepare the Cluster for Windows Nodes

```bash
# Ensure cluster uses a Windows-compatible network plugin
# RKE2 with Flannel (host-gw mode) works well for Windows

# Check current network plugin
kubectl get configmap rke2-cfg -n kube-system -o yaml | grep cni

# For new clusters, specify flannel in config
# /etc/rancher/rke2/config.yaml on control plane
cni: flannel
```

## Step 2: Prepare Windows Node

```powershell
# Run on Windows Server node (as Administrator)

# Check Windows version (must be 1809+)
[System.Environment]::OSVersion.Version
winver

# Enable required Windows features
Install-WindowsFeature -Name containers
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools

# Disable Windows Defender real-time monitoring
# (Can cause issues with container operations)
Set-MpPreference -DisableRealtimeMonitoring $true

# Disable Windows Firewall (configure proper firewall rules instead)
# or configure necessary ports:
# 6443 - Kubernetes API
# 10250 - kubelet
# 8472 - Flannel VXLAN
# 30000-32767 - NodePort range

# Set static IP or ensure DHCP is stable
# (IP changes can break Kubernetes node registration)
```

## Step 3: Install RKE2 Windows Agent

```powershell
# Download RKE2 Windows installer
$RKE2_VERSION = "v1.28.8+rke2r1"
Invoke-WebRequest `
  -Uri "https://github.com/rancher/rke2/releases/download/$RKE2_VERSION/rke2-windows-amd64.zip" `
  -OutFile "rke2-windows.zip"

Expand-Archive -Path rke2-windows.zip -DestinationPath C:\rke2

# Create configuration directory
New-Item -ItemType Directory -Force -Path C:\etc\rancher\rke2

# Create RKE2 agent configuration
@"
server: https://rke2-server.example.com:9345
token: my-cluster-token
node-label:
  - "kubernetes.io/os=windows"
  - "beta.kubernetes.io/os=windows"
"@ | Out-File -FilePath C:\etc\rancher\rke2\config.yaml -Encoding UTF8

# Install RKE2 as a Windows service
cd C:\rke2
.\rke2.exe agent service --add

# Start the service
Start-Service rke2
```

## Step 4: Verify Windows Node Registration

```bash
# From Linux control plane (kubectl)
# Check Windows node appears in cluster
kubectl get nodes -o wide

# Windows node should show:
# NAME          STATUS   ROLES    AGE   VERSION   OS-IMAGE
# win-node-01   Ready    <none>   5m    v1.28.x   Windows Server 2022

# Check node labels
kubectl describe node win-node-01 | grep -A 10 Labels

# Verify Windows node has correct OS labels
kubectl get nodes -l kubernetes.io/os=windows
```

## Step 5: Configure Windows-Specific Node Labels and Taints

```bash
# Add labels to Windows nodes for workload scheduling
kubectl label node win-node-01 \
  kubernetes.io/os=windows \
  node.kubernetes.io/windows-build=10.0.20348 \
  workload-type=windows

# Optionally taint Windows nodes so only Windows workloads schedule there
kubectl taint nodes win-node-01 \
  os=windows:NoSchedule

# Verify labels and taints
kubectl describe node win-node-01
```

## Step 6: Configure ImagePullSecrets for Windows

```bash
# Create registry secret that works for Windows containers
kubectl create secret docker-registry windows-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=winuser \
  --docker-password=password \
  --namespace=production

# Patch the default service account (or use imagePullSecrets in pod spec)
kubectl patch serviceaccount default \
  -n production \
  -p '{"imagePullSecrets": [{"name": "windows-registry-secret"}]}'
```

## Step 7: Verify Windows Container Runtime

```powershell
# On the Windows node, verify container runtime

# Check RKE2 status
Get-Service rke2

# Check containerd (used by RKE2 on Windows)
Get-Service containerd

# List running containers
& "C:\Program Files\containerd\ctr.exe" containers list

# Check node agent connectivity
Get-EventLog -LogName Application -Source rke2 -Newest 20
```

## Conclusion

Adding Windows worker nodes to Rancher creates a hybrid cluster capable of running both Linux and Windows containerized workloads from a single management plane. The key requirements are Windows Server 2019+ with containers feature enabled, a Windows-compatible network plugin (Flannel or Calico), and RKE2 Windows agent installation. After node registration, use node selectors and tolerations to direct Windows workloads to Windows nodes and Linux workloads to Linux nodes.
