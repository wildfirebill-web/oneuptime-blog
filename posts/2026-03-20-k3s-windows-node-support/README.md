# How to Configure K3s for Windows Node Support

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Windows, Kubernetes, Mixed Cluster, Node Selectors, Taints, SUSE Rancher

Description: Learn how to add Windows worker nodes to a K3s cluster for running Windows-native containerized workloads alongside Linux pods in a mixed operating system Kubernetes cluster.

---

K3s supports adding Windows Server nodes as worker nodes to an existing Linux-based K3s cluster. This enables running Windows-native applications (ASP.NET, .NET Framework) in containers alongside Linux workloads.

---

## Prerequisites

- K3s server running on Linux (version v1.25+)
- Windows Server 2019 or 2022 worker nodes
- The Windows nodes must be able to reach the K3s server on port 6443

---

## Step 1: Prepare Windows Nodes

On each Windows node (run in PowerShell as Administrator):

```powershell
# Enable Windows containers feature
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
Enable-WindowsOptionalFeature -Online -FeatureName containers -All

# Restart the node
Restart-Computer -Force
```

After reboot:

```powershell
# Install containerd for Windows
winget install --id=Microsoft.ContainerdForWindows --source winget

# Or manually
$version = "1.7.13"
curl.exe -L https://github.com/containerd/containerd/releases/download/v${version}/containerd-${version}-windows-amd64.tar.gz -o containerd.tar.gz
tar.exe xvf containerd.tar.gz
Copy-Item -Path ".\bin\*" -Destination "C:\Program Files\containerd" -Recurse -Force
```

---

## Step 2: Install K3s Agent on Windows

```powershell
# Set the K3s server address and token
$K3S_URL = "https://192.168.1.10:6443"
$K3S_TOKEN = "K1234567890abcdef..."

# Download K3s Windows agent
curl.exe -Lo k3s.exe https://github.com/k3s-io/k3s/releases/latest/download/k3s-arm64.exe

# Install as a Windows service
.\k3s.exe agent `
  --server $K3S_URL `
  --token $K3S_TOKEN `
  --node-label "kubernetes.io/os=windows" `
  --node-taint "os=windows:NoSchedule"
```

---

## Step 3: Verify Windows Node Registration

On the Linux server:

```bash
# Confirm Windows node appears
kubectl get nodes -o wide

# Nodes should show both linux and windows OS types
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.operatingSystem}{"\n"}{end}'
```

---

## Step 4: Deploy a Windows Workload

Use node selectors and tolerations to target Windows nodes:

```yaml
# windows-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: windows-app
  template:
    metadata:
      labels:
        app: windows-app
    spec:
      # Select Windows nodes only
      nodeSelector:
        kubernetes.io/os: windows

      # Tolerate the Windows taint
      tolerations:
        - key: os
          operator: Equal
          value: windows
          effect: NoSchedule

      containers:
        - name: app
          # Use a Windows Server Core base image
          image: mcr.microsoft.com/windows/servercore:ltsc2022
          command: ["powershell.exe", "-Command", "while($true) { Start-Sleep 30 }"]
          resources:
            requests:
              cpu: "0.5"
              memory: 512Mi
```

---

## Step 5: Ensure Linux-Only Pods Stay on Linux

Add a node selector to existing Linux workloads to prevent them from being scheduled to Windows nodes:

```yaml
nodeSelector:
  kubernetes.io/os: linux
```

Or add a default taint to Windows nodes that Linux pods won't tolerate:

```bash
kubectl taint node windows-node-01 os=windows:NoSchedule
```

---

## Best Practices

- Use **taints and tolerations** rather than node selectors alone to prevent accidental Linux pod scheduling on Windows nodes.
- Keep the Windows nodes in their own node pool and apply resource quotas via Rancher Projects.
- Be aware that some Kubernetes features (e.g., `hostNetwork: true`) behave differently on Windows nodes.
- Use Windows base images from Microsoft's official registry (`mcr.microsoft.com`) for security-patched base layers.
