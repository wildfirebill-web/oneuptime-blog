# How to Install Portainer CE on RHEL with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, RHEL, Red-hat, Docker, Installation, Enterprise-linux

Description: A guide to installing Portainer Community Edition on Red Hat Enterprise Linux (RHEL) 8 and 9 with Docker CE.

## Overview

Red Hat Enterprise Linux (RHEL) is one of the most widely used enterprise Linux distributions. While RHEL ships with Podman as its default container runtime, Docker CE can be installed alongside it. This guide covers installing Docker CE and Portainer CE on RHEL 8 and RHEL 9, including SELinux and subscription considerations.

## Prerequisites

- RHEL 8.x or 9.x (with or without active subscription)
- Minimum: 2GB RAM, 20GB disk
- Root or sudo access
- Internet connectivity or access to Docker CE mirror

## Step 1: Update System

```bash
sudo dnf update -y
```

## Step 2: Remove Conflicting Packages

RHEL ships with Podman, which conflicts with Docker:

```bash
# Check for conflicting packages

rpm -qa | grep -E 'docker|podman|containerd'

# Remove podman and related tools (optional)
sudo dnf remove -y podman podman-docker buildah skopeo 2>/dev/null

# Verify removal
rpm -qa | grep podman
```

## Step 3: Install Docker CE

Docker provides a CentOS-compatible repository that works on RHEL:

```bash
# Install prerequisites
sudo dnf install -y dnf-plugins-core

# Add Docker CE repository (uses CentOS repo, compatible with RHEL)
sudo dnf config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker CE
sudo dnf install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

# Enable and start Docker
sudo systemctl enable --now docker

# Add current user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

## Step 4: Configure SELinux

RHEL enforces SELinux by default:

```bash
# Check SELinux status
getenforce
# Output: Enforcing

# Docker's SELinux policy is installed automatically
# Verify docker SELinux policy
rpm -qa | grep container-selinux

# For Portainer, use the :z volume mount label
# This relabels the socket for container access
```

## Step 5: Configure firewalld

```bash
# Add Portainer ports to firewalld
sudo firewall-cmd --permanent --add-port=9443/tcp
sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-ports
```

## Step 6: Deploy Portainer CE

```bash
# Create data volume
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

# Verify
docker ps | grep portainer
docker logs portainer --tail 20
```

## Step 7: Access Portainer

```bash
# Get server IP
hostname -I | awk '{print $1}'
echo "Access Portainer at: https://$(hostname -I | awk '{print $1}'):9443"
```

Navigate to the URL, accept the self-signed certificate warning, and create your admin account.

## RHEL Subscription and Docker CE

Docker CE is not officially supported on RHEL with an active Red Hat subscription. Alternatives:

| Option | Notes |
|---|---|
| Docker CE (CentOS repo) | Works technically, not RH-supported |
| Podman + podman-docker | RH-supported, Docker API compatible |
| RHEL with Docker BE | Docker Inc. has an enterprise agreement |

For production RHEL environments requiring support contracts, consider Podman:

```bash
# Podman as Docker alternative (stays fully supported)
sudo dnf install -y podman podman-docker

# podman-docker provides a Docker-compatible CLI alias
docker ps  # Actually runs podman
```

## Verify Docker and Portainer

```bash
docker --version
docker info
curl -k https://localhost:9443/api/status
```

## Conclusion

Docker CE runs reliably on RHEL 8 and 9 despite not being officially supported by Red Hat. The SELinux `:z` volume label is critical for allowing Portainer to access the Docker socket. For organizations requiring full RHEL support, consider using Podman with the podman-docker compatibility layer as an alternative to Docker CE.
