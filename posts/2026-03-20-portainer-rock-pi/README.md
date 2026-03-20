# How to Run Portainer on a Rock Pi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Rock Pi, Rockchip, ARM64, Docker, Self-Hosted, Home Lab

Description: Install Docker and Portainer on Rock Pi single-board computers powered by Rockchip processors for a capable and affordable home lab platform.

## Introduction

Rock Pi boards from Radxa are powerful ARM64 single-board computers featuring Rockchip RK3399, RK3568, or RK3588 processors. The RK3588 in particular rivals desktop CPUs for single-board performance. This guide covers installing Docker and Portainer on Rock Pi boards running Ubuntu or Debian.

## Supported Models

- Rock Pi 4 (RK3399) — 4 TOPS NPU, 4GB LPDDR4
- Rock Pi X (Intel Atom) — x86_64 architecture
- Rock 4 SE (RK3399-T)
- Rock 5B (RK3588) — 6 TOPS NPU, up to 16GB RAM
- Rock 5A (RK3588S)

## Prerequisites

- Rock Pi board with Ubuntu 22.04 or Debian 11
- eMMC module or MicroSD (eMMC strongly recommended)
- SSH access

## Step 1: Flash and Configure the OS

Download the official Ubuntu image for your Rock Pi from Radxa's wiki. Flash using BalenaEtcher.

```bash
# SSH in with default credentials
ssh rock@<rock-pi-ip>  # or ssh radxa@<ip>

# Update system
sudo apt update && sudo apt full-upgrade -y

# Enable eMMC boot (if using eMMC)
# Follow Radxa's specific instructions for your model
```

## Step 2: Install Docker

```bash
# Install prerequisites
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg

# Add Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repo
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

sudo systemctl enable --now docker
```

## Step 3: Fix Rockchip cgroup Issues

Older Rockchip kernels may have incomplete cgroup support:

```bash
# Check cgroup support
cat /sys/fs/cgroup/cgroup.controllers

# If missing cpu or memory controllers, add kernel parameters
sudo nano /boot/firmware/cmdline.txt
# Add: cgroup_enable=memory cgroup_memory=1 cgroup_enable=cpuset

# For Rock Pi 5 with mainline kernel
sudo nano /boot/extlinux/extlinux.conf
# Add to APPEND line: cgroup_enable=memory cgroup_memory=1

sudo reboot
```

## Step 4: Configure Docker for Rockchip

```bash
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "features": {
    "buildkit": true
  }
}
EOF

sudo systemctl restart docker
```

## Step 5: Install Portainer

```bash
docker volume create portainer_data

docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 6: Configure Firewall

```bash
sudo ufw allow 9000/tcp comment 'Portainer HTTP'
sudo ufw allow 9443/tcp comment 'Portainer HTTPS'
sudo ufw allow ssh
sudo ufw enable
```

## Rock Pi 5 with RK3588 — Performance Stack

The Rock 5B with RK3588 supports running much heavier workloads. Example high-performance stack:

```yaml
version: "3.8"

services:
  # Database server - takes advantage of RK3588's performance
  postgresql:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: securepassword
      POSTGRES_DB: production
    volumes:
      - postgres_data:/var/lib/postgresql/data
    # Tune for RK3588 performance
    command: >
      postgres
      -c max_connections=200
      -c shared_buffers=1GB
      -c effective_cache_size=3GB
      -c maintenance_work_mem=256MB
    ports:
      - "5432:5432"
    restart: unless-stopped

  # Redis cache
  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    restart: unless-stopped

volumes:
  postgres_data:
```

## NPU Acceleration (Rock Pi 5 / RK3588)

The RK3588's 6 TOPS NPU can be accessed in containers:

```bash
# Check NPU device
ls /dev/rknpu*

# Docker run with NPU access
docker run -d \
  --device /dev/rknpu0:/dev/rknpu0 \
  --name rknn-app \
  your-rknn-image:latest
```

## Conclusion

Rock Pi boards with Portainer offer excellent home lab capabilities, especially the Rock 5B with RK3588 which provides near-desktop performance at low power consumption. The ARM64 architecture ensures compatibility with all major Docker images. For demanding workloads, the RK3588's combination of A76 performance cores and a capable NPU makes it stand out among SBCs.
