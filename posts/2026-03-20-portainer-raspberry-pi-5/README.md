# How to Install Portainer on Raspberry Pi 5

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Raspberry-pi-5, ARM64, Docker, Home-lab

Description: A guide to installing Portainer CE on the Raspberry Pi 5, taking advantage of its improved performance for running multiple containers.

## Overview

The Raspberry Pi 5 offers significantly better performance than its predecessors with a faster processor, more RAM options, and PCIe support for NVMe SSDs. This makes it an excellent platform for Portainer CE and a small home lab Docker environment. This guide covers installing Portainer CE on Raspberry Pi OS for the Pi 5.

## Prerequisites

- Raspberry Pi 5 (4GB or 8GB RAM)
- MicroSD card (32GB+) or NVMe SSD via PCIe HAT (recommended)
- Raspberry Pi OS 64-bit (Bookworm)
- Active cooling solution (Pi 5 runs hotter than Pi 4)

## Step 1: Flash and Boot Raspberry Pi OS

Use Raspberry Pi Imager to flash Raspberry Pi OS (Bookworm, 64-bit) to your storage device. Enable SSH in the advanced settings.

## Step 2: Update System

```bash
ssh pi@raspberrypi.local
sudo apt-get update && sudo apt-get upgrade -y
sudo reboot
```

## Step 3: Install Docker

```bash
# Docker installation script

curl -fsSL https://get.docker.com | sudo sh

# Add user to docker group
sudo usermod -aG docker pi
newgrp docker

# Enable Docker
sudo systemctl enable --now docker

# Verify
docker --version
docker info | grep Architecture
# Architecture: aarch64
```

## Step 4: Configure NVMe SSD Storage (Pi 5 Feature)

If using a PCIe NVMe HAT:

```bash
# Check if NVMe drive is detected
lsblk
# Should show nvme0n1

# Partition and format
sudo parted /dev/nvme0n1 mklabel gpt
sudo parted /dev/nvme0n1 mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/nvme0n1p1

# Mount for Docker storage
sudo mkdir -p /nvme
sudo mount /dev/nvme0n1p1 /nvme

# Add to fstab for persistence
echo '/dev/nvme0n1p1 /nvme ext4 defaults 0 2' | sudo tee -a /etc/fstab

# Configure Docker to use NVMe
cat > /etc/docker/daemon.json << 'EOF'
{
  "data-root": "/nvme/docker"
}
EOF
sudo systemctl restart docker
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
```

## Step 6: Performance Comparison vs Pi 4

The Pi 5 offers significant improvements for container workloads:

| Feature | Pi 4 (8GB) | Pi 5 (8GB) |
|---|---|---|
| CPU | Cortex-A72, 1.8GHz | Cortex-A76, 2.4GHz |
| CPU Performance | Baseline | ~2-3x faster |
| RAM Speed | LPDDR4X | LPDDR4X (faster) |
| Storage | MicroSD/USB | PCIe NVMe support |
| Container startup | ~5s | ~2s |
| Portainer UI load | ~3s | ~1s |

## Step 7: Thermal Management

The Pi 5 requires active cooling:

```bash
# Monitor temperature
watch -n 1 vcgencmd measure_temp

# Check CPU frequency (should stay at max if cooling is adequate)
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# Check throttling
vcgencmd get_throttled
# 0x0 = no throttling
```

## Accessing Portainer

```bash
echo "Portainer URL: https://$(hostname -I | awk '{print $1}'):9443"
```

## Conclusion

The Raspberry Pi 5 is a substantial upgrade for home lab Docker deployments. The improved CPU performance means containers start faster, Portainer's UI is more responsive, and you can run more containers simultaneously. The optional NVMe SSD eliminates SD card wear as a concern. Portainer CE makes the Pi 5 an accessible and capable home lab management platform.
