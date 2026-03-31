# How to Deploy Windows Workloads on RKE2

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Rancher, Window, Workloads, Multi-OS

Description: A complete guide to setting up Windows worker nodes in an RKE2 cluster and deploying Windows-based containerized workloads.

## Introduction

RKE2 supports mixed Linux/Windows clusters, allowing you to run Windows Server containers alongside Linux containers. This is valuable for organizations migrating .NET Framework applications to Kubernetes or running Windows-specific workloads while maintaining a unified cluster management experience.

## Prerequisites

- RKE2 Linux server nodes (control plane must run on Linux)
- Windows Server 2019 or 2022 worker nodes
- Calico CNI (required for Windows support in RKE2)
- All nodes on the same L2 network or with proper routing

## Architecture Overview

In a mixed cluster:
- Control plane nodes run on Linux only
- Worker nodes can be Linux or Windows
- The CNI plugin (Calico) provides cross-OS pod networking
- Windows nodes run `rke2-windows-agent`

## Step 1: Configure the Linux Server for Windows Support

On your Linux server node, configure RKE2 to use Calico CNI (required for Windows):

```yaml
# /etc/rancher/rke2/config.yaml on Linux server

token: "SharedClusterToken"
tls-san:
  - 192.168.1.100
  - my-cluster.example.com

# Calico is required for Windows node support
cni: calico
```

```bash
# Install and start RKE2 server
curl -sfL https://get.rke2.io | sudo sh -
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
```

## Step 2: Prepare the Windows Node

On the Windows Server node, run the following in an elevated PowerShell session:

```powershell
# Enable Windows Containers feature
Install-WindowsFeature -Name Containers -Restart

# After restart, verify Windows version (must be 2019 or 2022)
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion

# Download the RKE2 Windows agent installer
$RKE2Version = "v1.28.8+rke2r1"
Invoke-WebRequest -Uri "https://github.com/rancher/rke2/releases/download/$RKE2Version/rke2-windows-amd64.exe" `
    -OutFile "C:\rke2-windows-amd64.exe"
```

## Step 3: Install RKE2 on Windows

```powershell
# Create the RKE2 configuration directory
New-Item -ItemType Directory -Force -Path "C:\etc\rancher\rke2"

# Create the agent configuration file
$config = @"
server: https://192.168.1.100:9345
token: "SharedClusterToken"
"@
Set-Content -Path "C:\etc\rancher\rke2\config.yaml" -Value $config

# Run the RKE2 Windows installer
C:\rke2-windows-amd64.exe install

# Start the RKE2 Windows service
Start-Service rke2

# Enable the service to start on boot
Set-Service -Name rke2 -StartupType Automatic
```

## Step 4: Verify the Windows Node Joined

```bash
# From a Linux node with kubectl access
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

# Check that the Windows node appears
kubectl get nodes -o wide

# You should see the Windows node with OS=windows
kubectl get node <windows-node-name> -o jsonpath='{.status.nodeInfo.operatingSystem}'
```

## Step 5: Label and Taint the Windows Node

Label Windows nodes to allow workload targeting:

```bash
# Add a label to identify Windows nodes
kubectl label node <windows-node-name> kubernetes.io/os=windows

# Optionally add a taint to prevent Linux workloads from being scheduled here
kubectl taint node <windows-node-name> os=windows:NoSchedule
```

## Step 6: Deploy a Windows Workload

Windows containers must target Windows nodes using `nodeSelector` and tolerations.

```yaml
# windows-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-iis
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: windows-iis
  template:
    metadata:
      labels:
        app: windows-iis
    spec:
      # Target Windows nodes only
      nodeSelector:
        kubernetes.io/os: windows
      # Tolerate the Windows taint we added
      tolerations:
        - key: "os"
          operator: "Equal"
          value: "windows"
          effect: "NoSchedule"
      containers:
        - name: iis
          # Use Windows Server Core IIS image
          image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

```bash
kubectl apply -f windows-deployment.yaml

# Expose the deployment
kubectl expose deployment windows-iis --port=80 --type=LoadBalancer

# Monitor the pod startup (Windows containers take longer)
kubectl get pods -w -l app=windows-iis
```

## Step 7: Verify Windows Pod Networking

```powershell
# From the Windows node, verify the Calico HNS network was created
Get-HnsNetwork

# Check that the RKE2 Windows service is running
Get-Service rke2

# View the RKE2 Windows agent logs
Get-EventLog -LogName Application -Source rke2 -Newest 50
```

## Considerations and Limitations

1. **Image compatibility**: Windows container images must match the host OS version. An image built for Windows Server 2019 will not run on a 2022 host without `--isolation=hyperv`.

2. **No host network**: Windows pods cannot use `hostNetwork: true`.

3. **No privileged containers**: Windows does not support privileged containers.

4. **Named pipe volumes**: Windows pods can use named pipe volumes for IPC.

5. **Resource overhead**: Windows nodes have higher base resource consumption than Linux nodes.

## Conclusion

Deploying Windows workloads on RKE2 enables hybrid application deployments that span both Linux and Windows containers within a single cluster. The key requirements are using Calico CNI, running Windows Server 2019 or 2022 on worker nodes, and using `nodeSelector` in your pod specs to target the correct OS. With this setup, organizations can modernize Windows applications incrementally while managing them through a unified Kubernetes control plane.
