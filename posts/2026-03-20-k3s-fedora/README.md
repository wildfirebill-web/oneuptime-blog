# How to Install K3s on Fedora

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Fedora, Linux, Installation

Description: A guide to installing K3s on Fedora Linux, addressing SELinux, firewalld, and cgroup v2 requirements.

## Introduction

Fedora is a cutting-edge Linux distribution that often ships with the latest kernel features. Installing K3s on Fedora is straightforward but requires attention to Fedora-specific configurations: SELinux policies, firewalld rules, and cgroup v2 (which Fedora enables by default). This guide covers all the necessary steps for a successful K3s installation on Fedora.

## Tested Versions

- Fedora 37, 38, 39, 40
- K3s v1.26+

## Step 1: Update the System

```bash
# Update all packages
sudo dnf update -y

# Install useful tools
sudo dnf install -y curl wget git vim htop

# Check the Fedora version
cat /etc/fedora-release
uname -r
```

## Step 2: Configure Firewalld

Fedora uses firewalld by default. Open the ports K3s needs:

```bash
# Open K3s API server port
sudo firewall-cmd --permanent --add-port=6443/tcp

# Open port for K3s agent registration
sudo firewall-cmd --permanent --add-port=10250/tcp

# Flannel VXLAN (if using Flannel backend)
sudo firewall-cmd --permanent --add-port=8472/udp

# Flannel Wireguard (if using Wireguard backend)
sudo firewall-cmd --permanent --add-port=51820/udp
sudo firewall-cmd --permanent --add-port=51821/udp

# NodePort range
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/udp

# Allow traffic through CNI bridge
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --permanent --zone=trusted --add-interface=cni0
sudo firewall-cmd --permanent --zone=trusted --add-interface=flannel.1

# Reload firewalld
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-all
```

## Step 3: Configure SELinux

Fedora ships with SELinux in enforcing mode. K3s requires SELinux policies:

```bash
# Check SELinux status
getenforce
# Output: Enforcing

# Option 1: Install K3s SELinux policy (recommended)
sudo dnf install -y container-selinux selinux-policy-base

# The K3s install script can also install the SELinux RPM automatically
# (covered in Step 5)

# Option 2: Set SELinux to permissive mode (not recommended for production)
# sudo setenforce 0
# sudo sed -i 's/^ENFORCING=enforcing$/ENFORCING=permissive/' /etc/selinux/config
```

## Step 4: Disable Swap

```bash
# Check if swap is active
free -h

# Disable all swap
sudo swapoff -a

# Comment out swap entries in /etc/fstab to persist across reboots
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Verify
free -h
```

## Step 5: Configure cgroup v2

Fedora uses cgroup v2 by default. Verify K3s compatibility:

```bash
# Check cgroup version
stat -fc %T /sys/fs/cgroup/
# "cgroup2fs" = cgroup v2
# "tmpfs" = cgroup v1

# K3s v1.21+ supports cgroup v2 natively
# If using an older K3s, enable hybrid mode or force cgroup v1

# To force cgroup v1 (if needed for older K3s versions):
# sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"
# sudo reboot

# For K3s v1.21+, cgroup v2 works out of the box
```

## Step 6: Configure sysctl Settings

```bash
# Enable IP forwarding and bridge netfilter
sudo tee /etc/sysctl.d/k3s.conf > /dev/null <<EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Load br_netfilter module
sudo modprobe br_netfilter

# Make module loading persistent
echo "br_netfilter" | sudo tee /etc/modules-load.d/k3s.conf

# Apply sysctl settings
sudo sysctl --system
```

## Step 7: Install K3s

```bash
# Create configuration directory
sudo mkdir -p /etc/rancher/k3s

# Create K3s configuration
sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
token: "FedoraK3sToken"
tls-san:
  - $(hostname -I | awk '{print $1}')
  - $(hostname)
  - k3s.example.com
kubelet-arg:
  - "max-pods=110"
EOF

# Install K3s with SELinux support
# The INSTALL_K3S_SELINUX_WARN=true flag ensures SELinux policy RPM is installed
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_SELINUX_WARN=true \
    sudo sh -

# Check service status
sudo systemctl status k3s
```

## Step 8: Configure kubectl

```bash
# Set up kubeconfig for the current user
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Verify
kubectl get nodes
kubectl get pods --all-namespaces
```

## Step 9: Add an Agent Node on Fedora

```bash
# On the Fedora agent node, perform steps 1-6 first, then:
sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
server: "https://SERVER_IP:6443"
token: "FedoraK3sToken"
EOF

# Install K3s agent with SELinux support
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_EXEC="agent" \
    INSTALL_K3S_SELINUX_WARN=true \
    sudo sh -

sudo systemctl status k3s-agent
```

## Step 10: Deploy and Test

```bash
# Deploy a test workload
kubectl create deployment test-nginx --image=nginx:alpine
kubectl expose deployment test-nginx --port=80 --type=NodePort

# Get the NodePort
kubectl get svc test-nginx
NODE_PORT=$(kubectl get svc test-nginx -o jsonpath='{.spec.ports[0].nodePort}')

# Allow the NodePort through firewall
sudo firewall-cmd --permanent --add-port=${NODE_PORT}/tcp
sudo firewall-cmd --reload

# Test
curl http://localhost:$NODE_PORT

# Clean up
kubectl delete deployment test-nginx
kubectl delete svc test-nginx
```

## Troubleshooting Fedora-Specific Issues

### SELinux Denials

```bash
# Check for SELinux denials related to K3s
sudo ausearch -m AVC -ts recent | grep k3s

# Generate a local policy to allow denials (development only)
sudo ausearch -m AVC -ts recent | audit2allow -M k3s-local
sudo semodule -i k3s-local.pp
```

### firewalld Blocking Pod Traffic

```bash
# If pods can't communicate, add the pod CIDR to the trusted zone
sudo firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
sudo firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16
sudo firewall-cmd --reload
```

### cgroup v2 OOM Issues

```bash
# If pods are getting OOM-killed unexpectedly, check cgroup memory limits
systemd-cgls
# Look for memory.max settings
```

## Conclusion

K3s runs well on Fedora with proper SELinux policy installation and firewalld configuration. The main Fedora-specific considerations are opening firewall ports, ensuring the K3s SELinux RPM is installed, and verifying cgroup v2 compatibility with your K3s version. Fedora's up-to-date kernel benefits K3s with improved cgroup v2 performance and modern networking features, making it a capable K3s host for both development and production use.
