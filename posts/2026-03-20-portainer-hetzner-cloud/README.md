# How to Deploy Portainer on Hetzner Cloud - Part 3

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Hetzner, Cloud, Docker, Self-Hosted, DevOps, Europe

Description: Deploy Portainer on Hetzner Cloud for the best price-to-performance ratio in European cloud hosting, with Hetzner's firewall and volume storage.

## Introduction

Hetzner Cloud offers some of the best price-to-performance ratios in the cloud hosting market, with European data centers and excellent network performance. Their CX22 instance (2 vCPU, 4GB RAM) at ~€4/month is one of the most cost-effective options for running Portainer. This guide covers complete Portainer deployment on Hetzner Cloud.

## Prerequisites

- Hetzner Cloud account
- Hetzner Cloud CLI (`hcloud`) installed (optional)
- SSH key pair

## Step 1: Create a Hetzner Cloud Server

### Via Cloud Console

1. Log in to console.hetzner.cloud
2. Click **Create Server**
3. Configure:
   - **Location**: Choose EU or US datacenter
   - **Image**: Ubuntu 24.04
   - **Type**: CX22 (2 vCPU, 4GB RAM, 40GB SSD) - excellent value
   - **SSH Keys**: Add your SSH key
   - **Name**: `portainer-server`
4. Click **Create & Buy now**

### Via hcloud CLI

```bash
# Install hcloud

brew install hcloud  # macOS
# or: https://github.com/hetznercloud/cli

# Configure
hcloud context create portainer-project

# Create server
hcloud server create \
  --name portainer-server \
  --type cx22 \
  --image ubuntu-24.04 \
  --location fsn1 \
  --ssh-key "$(cat ~/.ssh/id_rsa.pub)"
```

## Step 2: Configure Hetzner Firewall

```bash
# Create firewall
hcloud firewall create --name portainer-fw

# Get your public IP
MY_IP=$(curl -s https://checkip.amazonaws.com)

# Add rules
hcloud firewall add-rule portainer-fw \
  --direction in \
  --protocol tcp \
  --port 22 \
  --source-ips ${MY_IP}/32

hcloud firewall add-rule portainer-fw \
  --direction in \
  --protocol tcp \
  --port 9000 \
  --source-ips ${MY_IP}/32

hcloud firewall add-rule portainer-fw \
  --direction in \
  --protocol tcp \
  --port 9443 \
  --source-ips ${MY_IP}/32

# Apply to server
hcloud firewall apply-to-resource portainer-fw \
  --type server \
  --server portainer-server
```

## Step 3: Install Docker

```bash
ssh root@<hetzner-server-ip>

# Update system
apt update && apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sh
systemctl enable docker
```

## Step 4: Attach Hetzner Volume

```bash
# Create volume
hcloud volume create \
  --name portainer-data \
  --size 50 \
  --server portainer-server \
  --format ext4

# Mount on server (Hetzner provides mount instructions after creation)
mkdir -p /mnt/HC_Volume_*
mount /dev/disk/by-id/scsi-0HC_Volume_* /mnt/HC_Volume_*

# Add to fstab for persistence
echo '/dev/disk/by-id/scsi-0HC_Volume_VOLUME_ID /mnt/portainer-data ext4 discard,nofail,defaults 0 0' >> /etc/fstab

# Configure Docker to use volume
mkdir -p /mnt/portainer-data/docker
tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "data-root": "/mnt/portainer-data/docker"
}
EOF
systemctl restart docker
```

## Step 5: Deploy Portainer

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

## Step 6: Configure Floating IP (Optional)

For stable IP across server replacements:

```bash
# Create Floating IP
hcloud floating-ip create --type ipv4 --home-location fsn1

# Assign to server
hcloud floating-ip assign FLOATING_IP_ID portainer-server

# Configure on server
ip addr add <floating-ip>/32 dev eth0
```

## Step 7: Enable Backups

Hetzner backups store up to 7 snapshots:

```bash
# Enable backups
hcloud server enable-backup portainer-server

# Or create manual snapshots
hcloud server create-image \
  --description "portainer-backup-$(date +%Y%m%d)" \
  portainer-server
```

## Conclusion

Hetzner Cloud provides exceptional value for Portainer deployments. The CX22 instance offers more RAM and better network performance than equivalently priced VMs at major cloud providers. Hetzner's European data centers are ideal for GDPR compliance, and the straightforward pricing with no surprise charges makes cost management predictable.
