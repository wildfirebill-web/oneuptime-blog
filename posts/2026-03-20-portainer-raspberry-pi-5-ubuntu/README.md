# How to Install Portainer on Raspberry Pi 5 with Ubuntu Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Raspberry Pi 5, Ubuntu, ARM64, Docker, Self-Hosted, Home Lab

Description: Set up Portainer on a Raspberry Pi 5 running Ubuntu Server 24.04 LTS to leverage the Pi 5's improved performance for a capable home lab server.

## Introduction

The Raspberry Pi 5 offers significantly improved performance over previous models with its Arm Cortex-A76 CPU and faster I/O. Pairing it with Ubuntu Server 24.04 LTS and Portainer creates a capable, energy-efficient home lab platform that can handle more demanding containerized workloads.

## Prerequisites

- Raspberry Pi 5 (4GB or 8GB recommended)
- MicroSD card (64GB+) or NVMe SSD via M.2 HAT (highly recommended)
- Ubuntu Server 24.04 LTS ARM64
- SSH access
- Active cooling (official Pi 5 case with fan recommended)

## Step 1: Install Ubuntu Server on Raspberry Pi 5

Use Raspberry Pi Imager to flash Ubuntu Server 24.04 LTS (64-bit) to your storage device. In the imager's OS customization:

- Set hostname
- Enable SSH
- Set username and password
- Configure Wi-Fi (if needed)

## Step 2: First Boot Setup

```bash
# SSH into the Pi

ssh ubuntu@<pi-ip>

# Update all packages
sudo apt update && sudo apt full-upgrade -y

# Install useful tools
sudo apt install -y curl wget git htop iotop net-tools

# Reboot to apply updates
sudo reboot
```

## Step 3: Configure NVMe SSD (Optional but Recommended)

If using the Pi 5 with M.2 HAT:

```bash
# Check NVMe is detected
lsblk | grep nvme

# Format and mount NVMe (adjust /dev/nvme0n1 as needed)
sudo mkfs.ext4 /dev/nvme0n1
sudo mkdir -p /data
echo '/dev/nvme0n1 /data ext4 defaults,noatime 0 2' | sudo tee -a /etc/fstab
sudo mount -a

# Move Docker data directory to NVMe
sudo mkdir -p /data/docker
```

## Step 4: Install Docker

```bash
# Install Docker using official script
curl -fsSL https://get.docker.com | sh

# Add current user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Configure Docker to use NVMe storage (if applicable)
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "data-root": "/data/docker",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
sudo systemctl enable docker
```

## Step 5: Install Portainer

```bash
# Create data volume
docker volume create portainer_data

# Deploy Portainer
docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 6: Configure Ubuntu Firewall (UFW)

```bash
# Allow Portainer ports
sudo ufw allow 9000/tcp comment 'Portainer HTTP'
sudo ufw allow 9443/tcp comment 'Portainer HTTPS'
sudo ufw allow ssh

# Enable UFW
sudo ufw enable

# Verify rules
sudo ufw status verbose
```

## Step 7: Configure Static IP

```bash
# Check current network interface
ip link show

# Configure netplan for static IP (Ubuntu 24.04)
sudo tee /etc/netplan/01-static.yaml > /dev/null << 'EOF'
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      routes:
        - to: default
          via: 192.168.1.1
EOF

sudo netplan apply
```

## Pi 5 Performance Tuning

### Enable Hardware Acceleration for Pi 5

```bash
# Install Pi 5 performance tools
sudo apt install -y rpi-utils

# Check CPU temperature
vcgencmd measure_temp

# Boost CPU governor for better container performance
echo 'performance' | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

### Persistent CPU Governor

```bash
sudo apt install -y cpufrequtils
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
```

## Monitoring Pi 5 Resources

```bash
# Check real-time stats
watch -n 1 'vcgencmd measure_temp && cat /sys/class/thermal/thermal_zone0/temp'

# Or deploy a monitoring container via Portainer
# Use the Netdata or Prometheus Node Exporter stack
```

## Conclusion

The Raspberry Pi 5 with Ubuntu Server and Portainer is one of the best value home lab platforms available. The improved CPU performance handles multiple concurrent containers smoothly, and Ubuntu Server's long-term support ensures security updates. An NVMe SSD dramatically improves I/O performance compared to MicroSD, making it suitable for database containers and other I/O-intensive workloads.
