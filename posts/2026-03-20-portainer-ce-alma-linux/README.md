# How to Install Portainer CE on AlmaLinux with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, almalinux, docker, installation, rhel-compatible

Description: A guide to installing Portainer Community Edition on AlmaLinux 8 and 9 with Docker, a RHEL-compatible distribution and popular CentOS replacement.

## Overview

AlmaLinux is a community-owned, RHEL-compatible Linux distribution created in response to CentOS's transition to CentOS Stream. It provides long-term stability and full binary compatibility with RHEL. This guide covers installing Docker CE and Portainer CE on AlmaLinux 8 and AlmaLinux 9.

## Prerequisites

- AlmaLinux 8.x or 9.x
- Minimum: 2GB RAM, 20GB disk
- Root or sudo access

## Step 1: Update System

```bash
sudo dnf update -y
```

## Step 2: Remove Conflicting Packages

```bash
# Remove any existing docker/podman packages
sudo dnf remove -y docker docker-common docker-selinux \
  docker-engine podman runc 2>/dev/null

# Clean up
sudo dnf autoremove -y
```

## Step 3: Install Docker CE

Docker provides a CentOS-compatible repository that works on AlmaLinux:

```bash
# Install dnf plugins
sudo dnf install -y dnf-plugins-core

# Add Docker repository (CentOS repo is compatible with AlmaLinux)
sudo dnf config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker CE packages
sudo dnf install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

# Start and enable Docker
sudo systemctl enable --now docker

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

## Step 4: Verify Docker Installation

```bash
docker --version
docker run hello-world
sudo systemctl status docker
```

## Step 5: Configure SELinux

AlmaLinux uses SELinux Enforcing mode by default:

```bash
# Check SELinux mode
getenforce
# Output: Enforcing

# Portainer needs the :z SELinux label for socket access
# This is handled in the docker run command below
```

## Step 6: Configure Firewall

```bash
# Open required ports
sudo firewall-cmd --permanent --add-port=9443/tcp
sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --reload

# Verify ports are open
sudo firewall-cmd --list-ports
```

## Step 7: Deploy Portainer CE

```bash
# Create Portainer data volume
docker volume create portainer_data

# Deploy Portainer CE with SELinux :z label
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock:z \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Check container status
docker ps | grep portainer
docker logs portainer --tail 20
```

## Step 8: Access and Initial Setup

```bash
# Get server IP
echo "Access Portainer at: https://$(hostname -I | awk '{print $1}'):9443"
```

1. Navigate to `https://<server-ip>:9443`
2. Accept the self-signed certificate warning
3. Create your admin username and password (minimum 12 characters)
4. Click **Create user** and begin managing containers

## AlmaLinux 8 vs AlmaLinux 9 Differences

| Feature | AlmaLinux 8 | AlmaLinux 9 |
|---|---|---|
| Default cgroups | cgroups v1 | cgroups v2 |
| SELinux | Enforcing | Enforcing |
| Python version | 3.6 | 3.9 |
| Docker compatibility | Full | Full |
| Support lifecycle | Until 2029 | Until 2032 |

```bash
# Check cgroup version on your system
stat -fc %T /sys/fs/cgroup/
# tmpfs = cgroups v1
# cgroup2fs = cgroups v2
```

## Optional: Enable Automatic Security Updates

```bash
sudo dnf install -y dnf-automatic

# Configure for security updates only
sudo sed -i 's/upgrade_type = default/upgrade_type = security/' \
  /etc/dnf/automatic.conf

sudo systemctl enable --now dnf-automatic.timer
```

## Conclusion

AlmaLinux is an excellent enterprise-grade platform for Portainer CE deployments. Its full RHEL binary compatibility ensures that enterprise Docker tooling works without modification. The SELinux `:z` volume label and firewalld configuration are standard requirements for all RHEL-compatible distributions. AlmaLinux's 10-year support lifecycle makes it a stable foundation for production Portainer deployments.
