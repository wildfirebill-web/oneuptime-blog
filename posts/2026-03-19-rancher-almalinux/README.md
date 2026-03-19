# How to Install Rancher on AlmaLinux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, AlmaLinux, Docker, Kubernetes, Installation

Description: A complete guide to installing Rancher on AlmaLinux 9 using Docker, including system hardening, Docker setup, and Rancher deployment.

AlmaLinux is a free, community-driven RHEL-compatible Linux distribution that emerged after CentOS shifted to a rolling release. It is governed by the AlmaLinux OS Foundation and is a popular choice for enterprise server deployments. This guide walks you through the full process of installing Rancher on AlmaLinux 9.

## Prerequisites

Before you begin, ensure you have:

- A server running AlmaLinux 9 with at least 4 GB RAM and 2 CPU cores
- Root or sudo access
- A static IP address or DNS name
- Internet access for downloading packages

## Step 1: Update the System

Update all packages:

```bash
sudo dnf update -y
```

## Step 2: Install EPEL Repository

Install the Extra Packages for Enterprise Linux repository for additional tools:

```bash
sudo dnf install -y epel-release
```

## Step 3: Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

## Step 4: Configure SELinux

Set SELinux to permissive mode:

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

## Step 5: Install Dependencies

```bash
sudo dnf install -y \
  yum-utils \
  device-mapper-persistent-data \
  lvm2 \
  curl \
  wget \
  tar \
  net-tools
```

## Step 6: Remove Conflicting Packages

AlmaLinux ships with Podman by default. Remove it to avoid conflicts:

```bash
sudo dnf remove -y podman buildah containers-common
```

## Step 7: Install Docker

Add the Docker CE repository. AlmaLinux uses the CentOS Docker repository:

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

Install Docker:

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Enable and start Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Add your user to the Docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify Docker:

```bash
docker --version
docker run hello-world
```

## Step 8: Configure the Firewall

Open the necessary ports:

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --reload
```

## Step 9: Load Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/rancher.conf
br_netfilter
overlay
EOF

sudo modprobe br_netfilter
sudo modprobe overlay
```

Configure network parameters:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-rancher.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

## Step 10: Configure Docker

Set up logging and storage:

```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl restart docker
```

## Step 11: Create Persistent Storage

```bash
sudo mkdir -p /opt/rancher
```

## Step 12: Run Rancher

Deploy Rancher:

```bash
docker run -d \
  --name rancher \
  --restart=unless-stopped \
  -p 80:80 \
  -p 443:443 \
  -v /opt/rancher:/var/lib/rancher \
  --privileged \
  rancher/rancher:latest
```

## Step 13: Get the Bootstrap Password

Wait about 60 seconds, then retrieve the password:

```bash
docker logs rancher 2>&1 | grep "Bootstrap Password:"
```

## Step 14: Access the Rancher UI

Open `https://<your-server-ip>` in your browser. Accept the self-signed certificate warning and log in with the bootstrap password.

Complete the initial setup:

1. Set a new admin password
2. Configure the Rancher server URL
3. Accept the terms and conditions

## AlmaLinux Specific Notes

**ELevate migration**: If you migrated from CentOS 7 or 8 to AlmaLinux 9 using ELevate, verify that all packages are properly updated and no legacy repositories remain:

```bash
dnf repolist
```

**Security profiles**: AlmaLinux supports OpenSCAP security profiles. If you have applied a security profile, you may need to create exceptions for Docker and Rancher:

```bash
sudo dnf install -y openscap-scanner scap-security-guide
```

**Automatic updates**: Consider enabling automatic security updates:

```bash
sudo dnf install -y dnf-automatic
sudo systemctl enable --now dnf-automatic-install.timer
```

## Monitoring the Installation

Check Rancher health:

```bash
# Container status
docker ps --filter name=rancher

# Resource usage
docker stats rancher --no-stream

# Detailed logs
docker logs rancher --tail 50
```

## Troubleshooting

```bash
# Check Docker service
sudo systemctl status docker
sudo journalctl -u docker --tail 50

# Check SELinux issues
sudo ausearch -m avc -ts recent
sudo sealert -a /var/log/audit/audit.log

# Check firewall
sudo firewall-cmd --list-all

# Check system resources
free -h
df -h
```

## Conclusion

You have successfully installed Rancher on AlmaLinux 9. AlmaLinux provides a stable, enterprise-grade platform with full RHEL compatibility and community-driven governance. Your Rancher instance is now ready to manage Kubernetes clusters across your infrastructure.
