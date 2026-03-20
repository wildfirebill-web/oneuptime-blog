# How to Install Portainer on Raspberry Pi 4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, raspberry-pi, arm64, docker, home-lab

Description: A step-by-step guide to installing Portainer CE on a Raspberry Pi 4 running Raspberry Pi OS (64-bit), perfect for home lab Docker management.

## Overview

The Raspberry Pi 4 with 4GB or 8GB RAM is an excellent platform for running Portainer CE in a home lab or small office environment. Portainer's ARM64 support makes it a natural fit for the Pi 4. This guide covers setting up Raspberry Pi OS 64-bit, installing Docker, and deploying Portainer CE.

## Prerequisites

- Raspberry Pi 4 (4GB or 8GB RAM recommended)
- MicroSD card (32GB+, Class 10 or better)
- Raspberry Pi OS (64-bit, Bookworm or Bullseye) — NOT 32-bit
- Internet connection (wired recommended for stability)

## Step 1: Flash Raspberry Pi OS 64-bit

Use Raspberry Pi Imager to flash the SD card:
1. Download Raspberry Pi Imager from https://www.raspberrypi.com/software/
2. Select "Raspberry Pi OS (64-bit)" — must be 64-bit for ARM64 Docker
3. In advanced settings, enable SSH and set hostname/password
4. Flash to MicroSD card and boot

## Step 2: Initial System Setup

```bash
# SSH into your Pi
ssh pi@raspberrypi.local

# Update system
sudo apt-get update && sudo apt-get upgrade -y
```

## Step 3: Install Docker on Raspberry Pi

```bash
# Use Docker's convenience installation script
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add pi user to docker group
sudo usermod -aG docker pi
newgrp docker

# Enable Docker on boot
sudo systemctl enable docker

# Verify Docker installation and architecture
docker --version
docker info | grep Architecture
# Architecture: aarch64 (ARM64)
```

## Step 4: Deploy Portainer CE for ARM64

Portainer provides native ARM64 images:

```bash
# Create data volume
docker volume create portainer_data

# Deploy Portainer CE (pulls ARM64 image automatically)
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Verify Portainer is running
docker ps | grep portainer
```

## Step 5: Access Portainer

```bash
hostname -I | awk '{print $1}'
# Navigate to: https://<pi-ip>:9443
```

## Performance Optimization for Raspberry Pi

```bash
# Monitor CPU temperature
vcgencmd measure_temp

# Monitor throttling
vcgencmd get_throttled
# 0x0 = not throttled

# Reduce GPU memory split (headless setup)
sudo raspi-config
# Advanced Options -> Memory Split -> 16
```

## Move Docker Storage to USB SSD

SD cards wear out quickly. Use a USB SSD for Docker data:

```bash
# After attaching and mounting USB SSD at /data:
sudo systemctl stop docker
sudo mv /var/lib/docker /data/docker
sudo ln -s /data/docker /var/lib/docker
sudo systemctl start docker
```

## Managing Home Lab Containers with Portainer

Use Portainer Stacks to manage multi-container apps:

```yaml
# Example docker-compose stack (paste in Portainer Stacks UI)
version: '3'
services:
  pihole:
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "80:80"
    volumes:
      - /etc/pihole:/etc/pihole
    restart: unless-stopped
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    ports:
      - "8123:8123"
    volumes:
      - /home/pi/homeassistant:/config
    restart: unless-stopped
    privileged: true
```

## Conclusion

The Raspberry Pi 4 is an excellent home lab platform for Portainer CE. Its ARM64 architecture is fully supported by Docker and Portainer. For the best experience, use 64-bit Raspberry Pi OS, store Docker data on a USB SSD rather than the MicroSD card, and ensure adequate cooling to prevent thermal throttling. Portainer transforms the Pi 4 into a capable home lab container management hub.
