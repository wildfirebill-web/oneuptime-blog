# How to Set Up RKE2 on ARM64 Architecture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Rancher, ARM64, Edge Computing, IoT

Description: Step-by-step guide to installing and configuring RKE2 on ARM64 hardware including cloud instances and embedded systems.

## Introduction

ARM64 (AArch64) processors are increasingly common in both cloud environments (AWS Graviton, Azure Ampere) and edge hardware (NVIDIA Jetson, Ampere Altra, Raspberry Pi 4). RKE2 provides native ARM64 binaries, making it straightforward to deploy a production-grade Kubernetes cluster on ARM64 hardware.

## Supported ARM64 Platforms

- AWS EC2 Graviton (t4g, m6g, c6g, r6g instances)
- Azure Ampere Altra instances
- NVIDIA Jetson AGX/NX series
- Raspberry Pi 4 Model B (8GB recommended for server)
- Rock Pi, Orange Pi, and other SBCs
- Apple Silicon (for development only)

## Prerequisites

- ARM64 hardware running Ubuntu 20.04+, RHEL 8+, or similar
- Minimum 2 vCPU and 4GB RAM for server nodes
- Minimum 1 vCPU and 2GB RAM for agent nodes
- Open ports: 9345, 6443, 10250

## Step 1: Verify Your Architecture

```bash
# Confirm the system is ARM64

uname -m
# Expected output: aarch64

# Check OS details
cat /etc/os-release
```

## Step 2: Prepare the System

```bash
# Update the system
sudo apt-get update && sudo apt-get upgrade -y   # Ubuntu/Debian
# or
sudo dnf update -y                               # RHEL/Fedora/Rocky

# Disable swap (required for Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load required kernel modules
sudo modprobe br_netfilter
sudo modprobe overlay

# Make kernel modules persistent
sudo tee /etc/modules-load.d/k8s.conf > /dev/null <<EOF
br_netfilter
overlay
EOF

# Configure sysctl for Kubernetes networking
sudo tee /etc/sysctl.d/k8s.conf > /dev/null <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## Step 3: Install RKE2 on ARM64

RKE2's install script automatically detects the architecture and downloads the correct binary.

```bash
# The install script auto-detects ARM64 and downloads the aarch64 binary
curl -sfL https://get.rke2.io | sudo sh -

# Verify the installed binary architecture
file /usr/local/bin/rke2
# Expected: ELF 64-bit LSB executable, ARM aarch64
```

### Installing a Specific Version

```bash
# Install a specific RKE2 version on ARM64
curl -sfL https://get.rke2.io | \
    INSTALL_RKE2_VERSION="v1.28.8+rke2r1" \
    sudo sh -
```

### Air-Gapped ARM64 Installation

```bash
# Download ARM64-specific artifacts
RKE2_VERSION="v1.28.8+rke2r1"

# Download the ARM64 binary
wget "https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2.linux-arm64.tar.gz"

# Download the ARM64 images
wget "https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2-images.linux-arm64.tar.zst"

# Install manually
sudo mkdir -p /var/lib/rancher/rke2/agent/images/
sudo cp rke2-images.linux-arm64.tar.zst /var/lib/rancher/rke2/agent/images/
sudo tar -xzf rke2.linux-arm64.tar.gz -C /usr/local/
```

## Step 4: Configure the RKE2 Server

```bash
sudo mkdir -p /etc/rancher/rke2

sudo tee /etc/rancher/rke2/config.yaml > /dev/null <<EOF
# RKE2 server configuration for ARM64
token: "Arm64ClusterToken"
tls-san:
  - $(hostname -I | awk '{print $1}')
  - $(hostname)

# Optimized for ARM64 (adjust based on your hardware)
kubelet-arg:
  - "max-pods=100"
  - "kube-reserved=cpu=200m,memory=512Mi"
  - "system-reserved=cpu=200m,memory=256Mi"
  - "eviction-hard=memory.available<200Mi,nodefs.available<10%"
EOF
```

## Step 5: Start the Server

```bash
# Enable and start the RKE2 server
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service

# Monitor the startup progress
sudo journalctl -u rke2-server -f

# Wait for the server to be ready (may take 1-3 minutes on ARM)
sudo /var/lib/rancher/rke2/bin/kubectl \
    --kubeconfig /etc/rancher/rke2/rke2.yaml \
    wait --for=condition=Ready node --all --timeout=300s
```

## Step 6: Configure kubectl

```bash
# Set up kubectl for the current user
mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Verify cluster access
kubectl get nodes -o wide

# Confirm the node reports ARM64 architecture
kubectl get node $(hostname) \
    -o jsonpath='{.status.nodeInfo.architecture}'
# Expected: arm64
```

## Step 7: Add ARM64 Agent Nodes

On additional ARM64 worker nodes:

```bash
# Get the server token
SERVER_TOKEN=$(sudo cat /var/lib/rancher/rke2/server/node-token)
SERVER_IP="192.168.1.100"

# Configure the agent
sudo mkdir -p /etc/rancher/rke2
sudo tee /etc/rancher/rke2/config.yaml > /dev/null <<EOF
server: https://${SERVER_IP}:9345
token: "${SERVER_TOKEN}"
EOF

# Install as agent
curl -sfL https://get.rke2.io | \
    INSTALL_RKE2_TYPE="agent" \
    sudo sh -

sudo systemctl enable rke2-agent.service
sudo systemctl start rke2-agent.service
```

## Step 8: Deploy ARM64-Compatible Workloads

Ensure your container images support ARM64. Use multi-arch images when available:

```yaml
# arm64-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-arm64
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          # Multi-arch image (supports both amd64 and arm64)
          image: nginx:alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
```

## Performance Considerations for ARM64

### CPU Governor

```bash
# Set the CPU governor to performance for better throughput
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

### Memory Pressure on 4GB Nodes

```yaml
# For nodes with only 4GB RAM, tighten resource reservations
kubelet-arg:
  - "max-pods=50"
  - "kube-reserved=cpu=300m,memory=768Mi"
  - "system-reserved=cpu=200m,memory=512Mi"
  - "eviction-hard=memory.available<150Mi,nodefs.available<8%"
```

## Conclusion

RKE2 provides first-class support for ARM64 architecture, making it an excellent choice for ARM cloud instances and edge hardware. The installation process is nearly identical to x86_64, with the install script automatically selecting the correct binaries. By following this guide, you can run a fully functional, production-grade Kubernetes cluster on ARM64 hardware, taking advantage of the superior performance-per-watt that ARM processors offer.
