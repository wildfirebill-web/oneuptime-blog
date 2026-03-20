# How to Configure K3s for Windows Node Support

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Windows, Mixed Cluster, DevOps, Containers

Description: Learn how to add Windows worker nodes to a K3s cluster to run Windows containerized workloads alongside Linux pods.

## Introduction

While K3s itself only runs on Linux, you can add Windows worker nodes to a K3s cluster to run Windows containers. This creates a mixed Linux/Windows cluster where Windows-specific workloads run on Windows nodes while Linux workloads run on Linux nodes. This is valuable for enterprises with .NET applications or Windows-specific services that need Kubernetes orchestration.

## Architecture

- **K3s Server**: Must run on Linux (the control plane)
- **Linux Agents**: Run Linux container workloads
- **Windows Agents**: Run Windows container workloads (Windows Server 2019/2022)

## Prerequisites

- Linux K3s server (existing cluster)
- Windows Server 2019 or 2022 nodes (not Windows 10/11 Home)
- Windows Subsystem for Linux 2 (WSL2) capabilities enabled
- Network connectivity between Windows nodes and K3s server on port 6443
- Hyper-V enabled on Windows nodes

## Step 1: Prepare Windows Nodes

On each Windows Server node:

```powershell
# Run as Administrator in PowerShell

# Check Windows Server version
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion

# Enable required Windows features
Enable-WindowsOptionalFeature -Online -FeatureName containers -All
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
Enable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -All
Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -All

# Reboot to apply
Restart-Computer -Force
```

After reboot:

```powershell
# Verify containers feature is enabled
Get-WindowsOptionalFeature -Online -FeatureName containers

# Check Hyper-V
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V
```

## Step 2: Install Container Runtime on Windows

Windows containers require either containerd or Docker:

```powershell
# Install containerd on Windows
# Download the latest release
$ContainerdVersion = "1.7.11"
Invoke-WebRequest `
  -Uri "https://github.com/containerd/containerd/releases/download/v$ContainerdVersion/containerd-$ContainerdVersion-windows-amd64.tar.gz" `
  -OutFile containerd.tar.gz

# Extract
tar -xvf containerd.tar.gz
New-Item -ItemType Directory -Path "C:\Program Files\containerd" -Force
Move-Item .\bin\* "C:\Program Files\containerd"

# Register containerd as a service
& "C:\Program Files\containerd\containerd.exe" config default |
  Out-File "C:\Program Files\containerd\config.toml" -Encoding ASCII

& "C:\Program Files\containerd\containerd.exe" --register-service
Start-Service containerd

# Verify containerd is running
Get-Service containerd
```

## Step 3: Install kubelet and kube-proxy for Windows

K3s's Windows agent uses the Kubernetes-native kubelet and kube-proxy on Windows:

```powershell
# Create directory for Kubernetes Windows binaries
New-Item -ItemType Directory -Path "C:\k" -Force

# Set Kubernetes version to match your K3s server
$K8sVersion = "v1.29.3"

# Download kubelet
Invoke-WebRequest `
  -Uri "https://dl.k8s.io/$K8sVersion/bin/windows/amd64/kubelet.exe" `
  -OutFile "C:\k\kubelet.exe"

# Download kube-proxy
Invoke-WebRequest `
  -Uri "https://dl.k8s.io/$K8sVersion/bin/windows/amd64/kube-proxy.exe" `
  -OutFile "C:\k\kube-proxy.exe"

# Add to PATH
$env:Path += ";C:\k"
[Environment]::SetEnvironmentVariable("Path", $env:Path, [EnvironmentVariableTarget]::Machine)
```

## Step 4: Configure Network Plugins on Windows

Windows requires specific CNI plugins:

```powershell
# Create CNI directory
New-Item -ItemType Directory -Path "C:\k\cni\config" -Force
New-Item -ItemType Directory -Path "C:\k\cni\bin" -Force

# Download Windows CNI plugins
$FlannelVersion = "0.23.0"
Invoke-WebRequest `
  -Uri "https://github.com/flannel-io/flannel/releases/download/v$FlannelVersion/flanneld-amd64.exe" `
  -OutFile "C:\k\cni\bin\flanneld.exe"

# Download wins (Windows service wrapper)
Invoke-WebRequest `
  -Uri "https://github.com/rancher/wins/releases/latest/download/wins.exe" `
  -OutFile "C:\k\wins.exe"

# Register wins as a service
& "C:\k\wins.exe" srv app run --register
Start-Service rancher-wins
```

## Step 5: Join Windows Node to K3s Cluster

```powershell
# Get the node token from the K3s server (run on Linux server)
# K3S_TOKEN=$(cat /var/lib/rancher/k3s/server/node-token)

# On the Windows node, configure the agent
$KubeNodeIP = (Get-NetIPAddress -AddressFamily IPv4 `
  -InterfaceAlias Ethernet).IPAddress

# Create kubelet configuration
@"
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
maxPods: 110
clusterDNS:
  - "10.43.0.10"
clusterDomain: "cluster.local"
"@ | Out-File "C:\k\kubelet-config.yaml"

# Start kubelet to join the cluster
& "C:\k\kubelet.exe" `
  --hostname-override $env:COMPUTERNAME `
  --kubeconfig "C:\k\kubeconfig" `
  --config "C:\k\kubelet-config.yaml" `
  --container-runtime-endpoint "npipe:////./pipe/containerd-containerd" `
  --node-labels "kubernetes.io/os=windows,role=windows-worker"
```

## Step 6: Deploy Windows Workloads

Windows pods must use `nodeSelector` to target Windows nodes:

```yaml
# windows-iis-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iis-server
  namespace: windows-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: iis
  template:
    metadata:
      labels:
        app: iis
    spec:
      # Required: schedule only on Windows nodes
      nodeSelector:
        kubernetes.io/os: windows

      # Required: tolerate Windows node taint (if added)
      tolerations:
        - key: "os"
          value: "windows"
          effect: "NoSchedule"

      containers:
        - name: iis
          # Use Windows Server Core base image
          image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022
          ports:
            - containerPort: 80
          resources:
            limits:
              memory: 2Gi
            requests:
              memory: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: iis-svc
  namespace: windows-apps
spec:
  selector:
    app: iis
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
```

## Step 7: Mixed Linux/Windows Cluster Configuration

Ensure Linux pods don't accidentally get scheduled on Windows nodes:

```bash
# On the K3s server (Linux), taint Windows nodes
kubectl taint nodes <windows-node-name> os=windows:NoSchedule

# Add OS label if not already set
kubectl label node <windows-node-name> kubernetes.io/os=windows
```

For Linux workloads, add a nodeSelector or affinity:

```yaml
spec:
  nodeSelector:
    kubernetes.io/os: linux  # Ensure Linux workloads stay on Linux nodes
```

## Step 8: Verify the Mixed Cluster

```bash
# Check all nodes (from Linux server)
kubectl get nodes -o wide
# Should show both Linux and Windows nodes

# Example output:
# NAME              STATUS   OS-IMAGE                    KERNEL-VERSION
# linux-server      Ready    Ubuntu 22.04 LTS            5.15.0-1045-azure
# linux-worker-01   Ready    Ubuntu 22.04 LTS            5.15.0-1045-azure
# windows-worker-01 Ready    Windows Server 2022          10.0.20348.2031

# Deploy test Windows container
kubectl apply -f windows-iis-deployment.yaml

# Check pods are scheduled on Windows nodes
kubectl get pods -n windows-apps -o wide
```

## Troubleshooting

```powershell
# Check kubelet logs on Windows
Get-EventLog -LogName Application -Source kubelet -Newest 20

# Check containerd status
Get-Service containerd

# View Windows container logs
& "C:\k\crictl.exe" --runtime-endpoint "npipe:////./pipe/containerd-containerd" ps
& "C:\k\crictl.exe" --runtime-endpoint "npipe:////./pipe/containerd-containerd" logs <container-id>
```

## Conclusion

Adding Windows nodes to K3s enables running Windows container workloads in a Kubernetes cluster while keeping the lightweight K3s control plane on Linux. This is valuable for .NET Framework applications, IIS-based services, and Windows-specific software that organizations are migrating to containers. The key is using `nodeSelector: kubernetes.io/os: windows` on Windows workloads and `kubernetes.io/os: linux` on Linux workloads to ensure proper scheduling in the mixed cluster.
