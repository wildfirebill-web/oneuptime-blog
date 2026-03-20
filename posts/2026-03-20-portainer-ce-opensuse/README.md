# How to Install Portainer CE on openSUSE with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, opensuse, docker, installation, suse, zypper

Description: A guide to installing Portainer Community Edition on openSUSE Leap and openSUSE Tumbleweed with Docker, covering zypper configuration and AppArmor considerations.

## Overview

openSUSE is a community distribution maintained by SUSE that underpins SUSE Linux Enterprise. It uses the `zypper` package manager and has AppArmor enabled by default. This guide covers installing Portainer CE on openSUSE Leap 15.5 and openSUSE Tumbleweed.

## Prerequisites

- openSUSE Leap 15.5 or Tumbleweed
- Minimum: 2GB RAM, 20GB disk
- Root or sudo access

## Step 1: Update System

```bash
# Update packages
sudo zypper refresh
sudo zypper update -y
```

## Step 2: Install Docker

```bash
# Install Docker from the official Docker repository
# Add Docker repository
sudo zypper addrepo https://download.docker.com/linux/sles/docker-ce.repo

# Or use the openSUSE community package
sudo zypper install -y docker

# Enable and start Docker
sudo systemctl enable --now docker

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

## Step 3: Configure AppArmor for Docker

openSUSE uses AppArmor which may need configuration for Docker:

```bash
# Check AppArmor status
sudo aa-status

# Docker installs its own AppArmor profile
# Verify Docker AppArmor profile is loaded
sudo aa-status | grep docker

# If not loaded:
sudo apparmor_parser -r /etc/apparmor.d/docker
```

## Step 4: Configure Firewall (SuSEfirewall2 or firewalld)

```bash
# For openSUSE using firewalld
sudo firewall-cmd --permanent --add-port=9443/tcp
sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-ports
```

## Step 5: Deploy Portainer CE

```bash
# Create data volume
docker volume create portainer_data

# Deploy Portainer CE
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Verify
docker ps | grep portainer
```

## Step 6: Access Portainer

```bash
# Get server IP
hostname -I | awk '{print $1}'
# Navigate to https://<ip>:9443
```

## Troubleshooting on openSUSE

### Docker Socket Permission Issue

```bash
# Check socket permissions
ls -la /var/run/docker.sock

# Ensure docker group exists and user is in it
getent group docker
groups $USER
```

### AppArmor Blocking Docker

```bash
# Check for AppArmor denials
sudo journalctl -k | grep apparmor | grep DENIED

# Temporarily put AppArmor in complain mode for testing
sudo aa-complain /usr/sbin/docker
```

## Using Docker CE from Official Docker Repository

For the latest Docker CE on openSUSE:

```bash
# Import Docker GPG key
sudo rpm --import https://download.docker.com/linux/sles/gpg

# Add repository (uses SLES packages which are compatible)
sudo zypper addrepo \
  https://download.docker.com/linux/sles/15/x86_64/stable/ \
  docker-ce-stable

sudo zypper refresh
sudo zypper install -y docker-ce docker-ce-cli containerd.io
```

## Conclusion

openSUSE's SUSE heritage makes it an excellent platform for enterprise-oriented Docker deployments. The AppArmor security framework provides additional container isolation. Once configured, Portainer CE runs seamlessly on openSUSE and provides the same management capabilities as on any other Linux distribution. openSUSE Leap's close alignment with SUSE Linux Enterprise makes it a solid choice for teams using Rancher in production.
