# How to Set Up a Home Lab with Portainer on Raspberry Pi - Homelab

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Raspberry Pi, Home Lab, Docker, Self-Hosted

Description: Learn how to build a complete self-hosted home lab using Portainer on a Raspberry Pi, running services like Pi-hole, Nginx Proxy Manager, and Home Assistant.

## Home Lab Architecture

```text
Raspberry Pi 4 (4GB+)
├── Portainer (management)
├── Pi-hole (DNS + ad blocking)
├── Nginx Proxy Manager (reverse proxy)
├── Home Assistant (home automation)
├── Nextcloud (file storage)
└── Grafana + Prometheus (monitoring)
```

## Hardware Requirements

- Raspberry Pi 4 (4GB or 8GB recommended) or Raspberry Pi 5
- 32GB+ microSD card (minimum) or USB3 SSD (recommended)
- Reliable power supply (official Pi power supply)
- Ethernet connection (recommended over Wi-Fi for server use)

## Step 1: Install Raspberry Pi OS

```bash
# Use Raspberry Pi Imager

# Select: Raspberry Pi OS Lite (64-bit) - no desktop needed

# Enable SSH in imager settings
# Set hostname, username, Wi-Fi (if needed)
```

## Step 2: Prepare the System

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y curl git vim

# Set static IP (optional but recommended)
sudo nano /etc/dhcpcd.conf
# Add:
# interface eth0
# static ip_address=192.168.1.100/24
# static routers=192.168.1.1
# static domain_name_servers=192.168.1.1
```

## Step 3: Install Docker

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh

# Add user to docker group
sudo usermod -aG docker $USER

# Enable Docker on boot
sudo systemctl enable docker

# Log out and back in, then verify
docker run hello-world
```

## Step 4: Install Portainer

```bash
docker volume create portainer_data

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Access: `https://192.168.1.100:9443`

## Step 5: Deploy Home Lab Services

In Portainer: **Stacks → Add Stack → homelab**

```yaml
version: "3.8"

services:
  pihole:
    image: pihole/pihole:latest
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8053:80/tcp"
    environment:
      - WEBPASSWORD=${PIHOLE_PASSWORD}
      - TZ=America/New_York
    volumes:
      - pihole_data:/etc/pihole
      - dnsmasq_data:/etc/dnsmasq.d
    cap_add:
      - NET_ADMIN

  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    restart: unless-stopped
    network_mode: host    # Required for mDNS/Bluetooth discovery
    environment:
      - TZ=America/New_York
    volumes:
      - ha_config:/config
      - /etc/localtime:/etc/localtime:ro

volumes:
  pihole_data:
  dnsmasq_data:
  ha_config:
```

## Step 6: Set Up Automatic Backups

```yaml
services:
  backup:
    image: offen/docker-volume-backup:latest
    restart: unless-stopped
    environment:
      - BACKUP_CRON_EXPRESSION=0 3 * * *    # Daily 3 AM
      - BACKUP_FILENAME=homelab-backup-%Y%m%d.tar.gz
      - BACKUP_RETENTION_DAYS=7
    volumes:
      - portainer_data:/backup/portainer_data:ro
      - pihole_data:/backup/pihole_data:ro
      - ha_config:/backup/ha_config:ro
      - /mnt/usb/backups:/archive
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

## Useful Home Lab Services

| Service | Image | Purpose |
|---------|-------|---------|
| Nginx Proxy Manager | jc21/nginx-proxy-manager | Reverse proxy with SSL |
| Vaultwarden | vaultwarden/server | Password manager |
| Nextcloud | nextcloud:latest | File storage |
| Jellyfin | jellyfin/jellyfin | Media server |
| Uptime Kuma | louislam/uptime-kuma | Service monitoring |
| Portainer | portainer/portainer-ce | Container management |

## Conclusion

A Raspberry Pi running Portainer is one of the most accessible home lab setups available. Portainer's web interface means you can manage all your self-hosted services from any device on your network, making it easy to add, update, and troubleshoot services without repeated SSH sessions.
