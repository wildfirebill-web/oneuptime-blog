# How to Install RKE2 on CentOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, CentOS, Installation, Rancher

Description: A step-by-step guide to installing RKE2 (Rancher Kubernetes Engine 2) on CentOS 7 and CentOS 8 for a production-ready Kubernetes cluster.

RKE2 provides a secure, compliant Kubernetes distribution that is well-suited for enterprise environments running CentOS. This guide covers the installation process on CentOS 7 and CentOS 8, including the specific prerequisites and configurations needed for these distributions.

## Prerequisites

- CentOS 7 (7.9+) or CentOS 8
- Minimum 2 vCPUs and 4 GB RAM per node
- Root or sudo access
- Network connectivity between nodes

## Step 1: Prepare the System

```bash
# Update system packages

sudo yum update -y

# Disable swap
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# Disable SELinux (or configure it properly - see RKE2 SELinux guide)
# For initial setup, disabling is simpler
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Load required kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/rke2.conf
overlay
br_netfilter
EOF

# Configure required sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/99-rke2.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## Step 2: Configure Firewall

```bash
# Configure firewalld for RKE2
sudo systemctl start firewalld
sudo systemctl enable firewalld

# Open required ports
sudo firewall-cmd --permanent --add-port=6443/tcp   # Kubernetes API
sudo firewall-cmd --permanent --add-port=9345/tcp   # RKE2 supervisor
sudo firewall-cmd --permanent --add-port=2379-2380/tcp # etcd
sudo firewall-cmd --permanent --add-port=10250/tcp  # Kubelet
sudo firewall-cmd --permanent --add-port=10257/tcp  # kube-controller-manager
sudo firewall-cmd --permanent --add-port=10259/tcp  # kube-scheduler
sudo firewall-cmd --permanent --add-port=8472/udp   # VXLAN
sudo firewall-cmd --permanent --add-port=51820/udp  # WireGuard
sudo firewall-cmd --permanent --add-masquerade

sudo firewall-cmd --reload
```

## Step 3: Install Required Dependencies

```bash
# Install required packages
sudo yum install -y \
  curl \
  wget \
  tar \
  git \
  conntrack \
  socat \
  nfs-utils

# For CentOS 7, install a newer version of iptables if needed
# RKE2 works with both legacy and nftables
```

## Step 4: Install RKE2 Server

```bash
# Download and install RKE2
curl -sfL https://get.rke2.io | sudo sh -

# Create the RKE2 configuration directory
sudo mkdir -p /etc/rancher/rke2/

# Create a basic server configuration
cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
# Bind the server to the node's IP
# node-ip: <NODE_IP>

# Optional: Specify the cluster DNS domain
cluster-dns: 10.43.0.10
cluster-domain: cluster.local

# Optional: configure additional SANs for the API server
# tls-san:
#   - my.dns.example.com
#   - 10.0.0.100
EOF

# Enable and start the RKE2 server
sudo systemctl enable rke2-server
sudo systemctl start rke2-server

# Monitor startup logs
sudo journalctl -u rke2-server -f &
```

## Step 5: Verify Installation

```bash
# Add RKE2 binaries to PATH
cat >> ~/.bashrc << 'EOF'
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
EOF

source ~/.bashrc

# Wait for RKE2 to be ready
sleep 30

# Check nodes
kubectl get nodes

# Check all pods
kubectl get pods -A
```

## Step 6: Install RKE2 Agent on Worker Nodes

```bash
# On worker nodes, get the server token first
# Run on the server node:
sudo cat /var/lib/rancher/rke2/server/node-token

# On each worker node:
# Install RKE2 agent
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sudo sh -

# Configure the agent
sudo mkdir -p /etc/rancher/rke2/

cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
# Server URL for the RKE2 control plane
server: https://<SERVER_IP>:9345
# Authentication token
token: <NODE_TOKEN>
EOF

# Start the agent
sudo systemctl enable rke2-agent
sudo systemctl start rke2-agent

# Watch logs for issues
sudo journalctl -u rke2-agent -f
```

## Step 7: Handle CentOS 7 Specific Issues

```bash
# CentOS 7 may need a newer kernel for eBPF support
# Check the current kernel version
uname -r

# If kernel is older than 5.4, update it for better compatibility
# For CentOS 7, you can use ELRepo
sudo yum install -y elrepo-release
sudo yum --enablerepo=elrepo-kernel install -y kernel-ml

# Update GRUB to boot into the new kernel
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo grub2-set-default 0

# Reboot to use the new kernel
sudo reboot
```

## Conclusion

Installing RKE2 on CentOS follows the same general process as other Linux distributions, with a few CentOS-specific considerations like firewalld configuration and potential kernel requirements. For CentOS 8 users, note that CentOS 8 reached EOL in December 2021, so consider migrating to Rocky Linux or AlmaLinux for continued support. Once RKE2 is installed, you can register the cluster with Rancher for centralized management.
