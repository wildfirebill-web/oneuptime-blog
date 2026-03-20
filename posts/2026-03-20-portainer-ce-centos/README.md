# How to Install Portainer CE on CentOS with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, centos, docker, installation, rhel

Description: A step-by-step guide to installing Portainer Community Edition on CentOS 7 and CentOS Stream 8/9 with Docker, including SELinux considerations.

## Overview

CentOS is widely used in enterprise environments. This guide covers installing Portainer CE on CentOS 7, CentOS Stream 8, and CentOS Stream 9, including Docker installation via the official Docker repository and handling CentOS-specific configurations like SELinux.

## Prerequisites

- CentOS 7, CentOS Stream 8, or CentOS Stream 9
- Minimum: 2GB RAM, 20GB disk
- Root or sudo access

## Step 1: Update System

```bash
# Update all packages
sudo yum update -y   # CentOS 7
# or
sudo dnf update -y   # CentOS Stream 8/9
```

## Step 2: Install Docker on CentOS

### CentOS 7

```bash
# Remove old Docker versions
sudo yum remove docker docker-common docker-selinux docker-engine 2>/dev/null

# Install prerequisites
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

# Add Docker repository
sudo yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker
sudo yum install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# Start and enable Docker
sudo systemctl enable --now docker
```

### CentOS Stream 8/9

```bash
# Remove old versions
sudo dnf remove docker docker-common docker-selinux docker-engine 2>/dev/null

# Install dnf-plugins-core
sudo dnf install -y dnf-plugins-core

# Add Docker repository
sudo dnf config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker
sudo dnf install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# Start and enable Docker
sudo systemctl enable --now docker

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

## Step 3: Configure SELinux

CentOS uses SELinux by default. Configure it properly for Docker:

```bash
# Check SELinux status
getenforce
# Options: Enforcing, Permissive, Disabled

# Option 1: Set SELinux to Permissive (easier but less secure)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Option 2: Keep SELinux Enforcing and use proper labels
# The :z mount option relabels the volume for Docker
# (used in the Portainer run command below)
```

## Step 4: Configure Firewall

```bash
# Open Portainer ports in firewalld
sudo firewall-cmd --permanent --add-port=9443/tcp
sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-all
```

## Step 5: Deploy Portainer CE

```bash
# Create data volume
docker volume create portainer_data

# Deploy Portainer (with SELinux label if SELinux is Enforcing)
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock:z \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Without SELinux :z label (if SELinux is Permissive or Disabled)
# docker run -d \
#   -p 8000:8000 \
#   -p 9443:9443 \
#   --name portainer \
#   --restart=always \
#   -v /var/run/docker.sock:/var/run/docker.sock \
#   -v portainer_data:/data \
#   portainer/portainer-ce:latest
```

## Step 6: Verify and Access

```bash
# Check Portainer is running
docker ps | grep portainer

# Check logs
docker logs portainer

# Access Portainer UI
echo "Access Portainer at: https://$(hostname -I | awk '{print $1}'):9443"
```

## SELinux Troubleshooting

If Portainer fails to start with SELinux in Enforcing mode:

```bash
# Check SELinux denials
sudo ausearch -c 'portainer' --raw | audit2why

# Create SELinux policy module to allow Portainer
sudo ausearch -c 'portainer' --raw | audit2allow -M portainer-policy
sudo semodule -i portainer-policy.pp

# Or temporarily set to permissive for testing
sudo setenforce 0
```

## Keeping Docker and Portainer Updated

```bash
# Update Docker
sudo yum update docker-ce     # CentOS 7
sudo dnf update docker-ce     # CentOS Stream 8/9

# Update Portainer
docker pull portainer/portainer-ce:latest
docker stop portainer && docker rm portainer
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock:z \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Conclusion

Installing Portainer CE on CentOS requires attention to SELinux configuration and firewalld rules that are not present in Debian-based systems. Using the `:z` flag on volume mounts ensures proper SELinux labeling for the Docker socket. Once running, Portainer works identically across all Linux distributions. For production CentOS deployments, consider using Rocky Linux or AlmaLinux as they receive longer community support than CentOS Stream.
