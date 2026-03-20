# How to Install RKE on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE, Kubernetes, Rancher, Installation, Linux

Description: A complete guide to installing Rancher Kubernetes Engine (RKE) on Linux and bootstrapping your first Kubernetes cluster.

## Introduction

Rancher Kubernetes Engine (RKE) is a CNCF-certified Kubernetes distribution that runs entirely within Docker containers. Unlike RKE2, it uses Docker as the container runtime and is managed by a standalone CLI binary. RKE is a popular choice for teams that prefer a traditional approach to Kubernetes cluster provisioning with full control over the configuration.

## Prerequisites

- Linux nodes (Ubuntu 18.04+, CentOS 7+, or RHEL 7+)
- Docker installed on all nodes
- SSH key-based access from the machine running RKE to all cluster nodes
- Minimum 2 vCPU and 4GB RAM per node

## Step 1: Install Docker on All Nodes

RKE requires Docker on every cluster node.

```bash
# Install Docker using the convenience script (for all nodes)
curl -fsSL https://get.docker.com | sudo sh

# Add your user to the docker group
sudo usermod -aG docker $USER

# Start and enable Docker
sudo systemctl enable docker
sudo systemctl start docker

# Verify Docker installation
docker --version
```

For specific Docker versions compatible with your Kubernetes version, check the RKE release notes.

## Step 2: Configure SSH Access

RKE connects to nodes via SSH. Set up key-based authentication from your workstation to all cluster nodes.

```bash
# Generate an SSH key pair (if you don't have one)
ssh-keygen -t ed25519 -C "rke-admin"

# Copy the public key to each cluster node
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@node1.example.com
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@node2.example.com
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@node3.example.com

# Test SSH connectivity
ssh user@node1.example.com docker ps
```

## Step 3: Download the RKE Binary

RKE is a single binary downloaded from GitHub Releases.

```bash
# Get the latest RKE release version
RKE_VERSION=$(curl -s https://api.github.com/repos/rancher/rke/releases/latest | grep tag_name | cut -d '"' -f 4)

# Download the RKE binary for Linux AMD64
curl -LO "https://github.com/rancher/rke/releases/download/${RKE_VERSION}/rke_linux-amd64"

# Make it executable
chmod +x rke_linux-amd64

# Move it to your PATH
sudo mv rke_linux-amd64 /usr/local/bin/rke

# Verify the installation
rke --version
```

## Step 4: Prepare Nodes

On each cluster node, configure the required kernel settings:

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Enable IPv4 forwarding
sudo tee /etc/sysctl.d/k8s.conf > /dev/null <<EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system

# Load required kernel modules
sudo modprobe br_netfilter
```

## Step 5: Create the Cluster Configuration

Generate a starter config or create it manually:

```bash
# Generate a sample cluster.yml
rke config --name cluster.yml
```

Or create a minimal `cluster.yml` manually:

```yaml
# cluster.yml
kubernetes_version: v1.28.8-rancher1-1

nodes:
  - address: 192.168.1.101
    user: ubuntu
    role:
      - controlplane
      - etcd
    ssh_key_path: ~/.ssh/id_ed25519
  - address: 192.168.1.102
    user: ubuntu
    role:
      - worker
    ssh_key_path: ~/.ssh/id_ed25519
  - address: 192.168.1.103
    user: ubuntu
    role:
      - worker
    ssh_key_path: ~/.ssh/id_ed25519

network:
  plugin: canal

services:
  etcd:
    backup_config:
      enabled: true
      interval_hours: 12
      retention: 6
```

## Step 6: Deploy the Cluster

```bash
# Deploy the RKE cluster (run from the directory containing cluster.yml)
rke up

# The output will show progress as each node is configured
# This process typically takes 5-15 minutes
```

RKE will:
1. Connect to all nodes via SSH
2. Deploy etcd, control plane, and worker components as Docker containers
3. Generate the `kube_config_cluster.yml` kubeconfig file

## Step 7: Configure kubectl

```bash
# Use the generated kubeconfig
export KUBECONFIG=$(pwd)/kube_config_cluster.yml

# Verify cluster access
kubectl get nodes

# Copy to your default kubeconfig location
mkdir -p ~/.kube
cp kube_config_cluster.yml ~/.kube/config
```

## Step 8: Verify the Installation

```bash
# Check all nodes are Ready
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Run a test workload
kubectl run nginx-test --image=nginx --restart=Never
kubectl get pod nginx-test
kubectl delete pod nginx-test
```

## Enabling the RKE Dashboard (Optional)

```bash
# Deploy the Kubernetes Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Create an admin service account
kubectl create serviceaccount admin-user -n kubernetes-dashboard
kubectl create clusterrolebinding admin-user \
    --clusterrole=cluster-admin \
    --serviceaccount=kubernetes-dashboard:admin-user

# Get the access token
kubectl -n kubernetes-dashboard create token admin-user

# Access the dashboard
kubectl proxy
# Open: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

## Conclusion

Installing RKE on Linux is straightforward: download the binary, prepare your nodes with Docker and SSH access, write a `cluster.yml` configuration file, and run `rke up`. Within minutes, you have a fully functional Kubernetes cluster. RKE's simplicity and the single-binary approach make it ideal for teams that want fast cluster provisioning without complex prerequisites. For new deployments, also consider RKE2 for enhanced security and compliance features.
