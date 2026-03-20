# How to Install Portainer CE on Rocky Linux with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Rocky-linux, Docker, Installation, Rhel-compatible

Description: A step-by-step guide to installing Portainer Community Edition on Rocky Linux 8 and 9 with Docker, the recommended CentOS replacement for enterprise environments.

## Overview

Rocky Linux is a RHEL-compatible distribution created after CentOS's transition to CentOS Stream. It provides long-term stability and is widely used as a CentOS replacement in enterprise environments. This guide covers installing Docker CE and Portainer on Rocky Linux 8 and Rocky Linux 9.

## Prerequisites

- Rocky Linux 8.x or 9.x
- Minimum: 2GB RAM, 20GB disk
- Root or sudo access

## Step 1: Update System

```bash
sudo dnf update -y
```

## Step 2: Install Docker CE

```bash
# Remove any conflicting packages

sudo dnf remove -y docker docker-common docker-selinux \
  docker-engine podman runc 2>/dev/null

# Install dnf plugins
sudo dnf install -y dnf-plugins-core

# Add Docker repository (uses CentOS repo, compatible with Rocky)
sudo dnf config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker CE
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

## Step 3: Verify Docker

```bash
docker --version
docker run hello-world
sudo systemctl status docker
```

## Step 4: Configure SELinux

Rocky Linux 9 has SELinux in Enforcing mode by default:

```bash
# Check SELinux status
getenforce

# For production, keep SELinux Enforcing and use :z mount label
# SELinux relabels the bind mount for container access
```

## Step 5: Configure firewalld

```bash
# Open Portainer ports
sudo firewall-cmd --permanent --add-port=9443/tcp
sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --reload

# Verify open ports
sudo firewall-cmd --list-ports
```

## Step 6: Deploy Portainer CE

```bash
# Create data volume
docker volume create portainer_data

# Deploy with SELinux :z label for Enforcing mode
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock:z \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 7: First Access

```bash
# Get server IP
hostname -I | awk '{print $1}'

echo "Access Portainer at: https://$(hostname -I | awk '{print $1}'):9443"
```

Navigate to the URL, accept the self-signed certificate, and create your admin account.

## Step 8: Enable Automatic Updates (dnf-automatic)

```bash
# Install dnf-automatic for automatic security updates
sudo dnf install -y dnf-automatic

# Configure for security updates only
sudo sed -i 's/upgrade_type = default/upgrade_type = security/' \
  /etc/dnf/automatic.conf

sudo systemctl enable --now dnf-automatic.timer
```

## Rocky Linux 8 vs Rocky Linux 9 Differences

| Feature | Rocky Linux 8 | Rocky Linux 9 |
|---|---|---|
| Default cgroups | cgroups v1 | cgroups v2 |
| SELinux | Enforcing | Enforcing |
| Python | 3.6 | 3.9 |
| Docker compatibility | Full | Full |

```bash
# Check cgroup version
cat /sys/fs/cgroup/cgroup.controllers
# If empty, using cgroups v1
# If contains entries, using cgroups v2
```

## Troubleshooting

### Docker Fails to Start with cgroups v2

```bash
# Rocky Linux 9 with cgroups v2 - ensure containerd is configured
sudo containerd config default > /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl restart docker
```

## Conclusion

Rocky Linux is an excellent CentOS replacement for enterprise Portainer deployments. Its binary compatibility with RHEL ensures that enterprise software designed for RHEL works without modification. The SELinux enforcement requires using `:z` volume labels, and firewalld needs explicit port rules - both are standard enterprise Linux practices. Rocky Linux's 10-year support lifecycle makes it a stable foundation for production Portainer deployments.
