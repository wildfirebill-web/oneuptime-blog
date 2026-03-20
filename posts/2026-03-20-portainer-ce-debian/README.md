# How to Install Portainer CE on Debian with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, debian, docker, installation, container-management

Description: A step-by-step guide to installing Portainer Community Edition on Debian Linux with Docker, covering Docker setup and Portainer initial configuration.

## Overview

Portainer CE provides a user-friendly web interface for managing Docker containers, images, volumes, and networks. This guide covers the complete process of installing Portainer CE on Debian 11 (Bullseye) and Debian 12 (Bookworm), including Docker installation from the official Docker repository.

## Prerequisites

- Debian 11 (Bullseye) or Debian 12 (Bookworm)
- Minimum: 2GB RAM, 20GB disk
- Root or sudo access
- Internet connectivity

## Step 1: Update System

```bash
# Update package index
sudo apt-get update && sudo apt-get upgrade -y

# Install required packages
sudo apt-get install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release
```

## Step 2: Install Docker on Debian

```bash
# Remove old Docker versions
sudo apt-get remove -y docker docker-engine docker.io containerd runc 2>/dev/null

# Add Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

# Enable and start Docker
sudo systemctl enable --now docker

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker   # Apply group change without logout
```

## Step 3: Verify Docker

```bash
# Verify Docker installation
docker --version
sudo systemctl status docker

# Run test container
docker run hello-world
```

## Step 4: Deploy Portainer CE

```bash
# Create persistent data volume
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
```

## Step 5: Open Firewall Ports (if using iptables/nftables)

```bash
# Check current firewall status
sudo iptables -L -n

# Allow Portainer ports
sudo iptables -A INPUT -p tcp --dport 9443 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 8000 -j ACCEPT

# Make firewall rules persistent
sudo apt-get install -y iptables-persistent
sudo netfilter-persistent save
```

## Step 6: Access Portainer

Navigate to `https://<server-ip>:9443` in your browser.

1. Accept the self-signed certificate warning
2. Set your admin username and password
3. Select "Get Started" to manage your local Docker environment

## Step 7: Configure Portainer to Start on Boot

Portainer is already configured to restart automatically via `--restart=always`. Verify:

```bash
# Check restart policy
docker inspect portainer --format='{{.HostConfig.RestartPolicy.Name}}'
# Output: always

# Verify Portainer starts after Docker service restart
sudo systemctl restart docker
sleep 5
docker ps | grep portainer
```

## Keeping Portainer Updated

```bash
# Update Portainer to latest version
docker pull portainer/portainer-ce:latest
docker stop portainer && docker rm portainer
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Troubleshooting on Debian

### Permission Denied on Docker Socket

```bash
# Check socket permissions
ls -la /var/run/docker.sock
# Expected: srw-rw---- root docker

# If permission issue, check group membership
groups $USER
# Should include 'docker'

# Apply without logout
newgrp docker
```

### Portainer Not Starting

```bash
# Check Portainer logs for errors
docker logs portainer

# Common error: port already in use
sudo netstat -tlnp | grep -E '9443|8000'

# Common error: volume mount issue
docker volume inspect portainer_data
```

## Conclusion

Installing Portainer CE on Debian follows the same pattern as Ubuntu with minor differences in the Docker repository configuration. Once installed, Portainer provides an intuitive management interface for your Debian-based Docker host. The data volume ensures your Portainer configuration persists through container updates and restarts.
