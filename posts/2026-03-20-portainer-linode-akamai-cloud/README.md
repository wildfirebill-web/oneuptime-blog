# How to Deploy Portainer on Linode/Akamai Cloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Linode, Akamai, Cloud, Docker, Self-Hosted, DevOps

Description: Deploy Portainer on a Linode (now Akamai Cloud) instance with firewall configuration, block storage, and optional NodeBalancer for production use.

## Introduction

Linode, now part of Akamai Cloud, offers competitive pricing with excellent network performance and straightforward Linux VMs. Its global network and developer-friendly tools make it popular for self-hosted applications. This guide covers deploying Portainer on a Linode instance with proper security configuration.

## Prerequisites

- Linode/Akamai account
- Linode CLI installed (optional)
- SSH key pair

## Step 1: Create a Linode Instance

### Via Linode Console

1. Click **Create Linode**
2. Choose:
   - **Image**: Ubuntu 24.04 LTS
   - **Region**: Closest datacenter
   - **Linode Plan**: Shared CPU > Linode 4GB (minimum recommended)
   - **Root Password**: Set a strong password
   - **SSH Keys**: Add your public key
   - **Label**: `portainer-linode`
3. Click **Create Linode**

### Via Linode CLI

```bash
# Install and configure CLI
pip3 install linode-cli
linode-cli configure

# Create Linode
linode-cli linodes create \
  --type g6-standard-2 \
  --region us-east \
  --image linode/ubuntu24.04 \
  --label portainer-linode \
  --authorized_keys "$(cat ~/.ssh/id_rsa.pub)" \
  --root_pass YourSecureRootPassword
```

## Step 2: Configure Cloud Firewall

```bash
# Via Linode CLI
linode-cli firewalls create \
  --label portainer-fw \
  --rules.inbound '[
    {"action":"ACCEPT","protocol":"TCP","ports":"22","addresses":{"ipv4":["YOUR.IP.HERE/32"]},"label":"allow-ssh"},
    {"action":"ACCEPT","protocol":"TCP","ports":"9000","addresses":{"ipv4":["YOUR.IP.HERE/32"]},"label":"allow-portainer-http"},
    {"action":"ACCEPT","protocol":"TCP","ports":"9443","addresses":{"ipv4":["YOUR.IP.HERE/32"]},"label":"allow-portainer-https"}
  ]' \
  --rules.inbound_policy DROP \
  --rules.outbound_policy ACCEPT
```

### Via Linode Console

1. Navigate to **Firewalls > Create Firewall**
2. Label: `portainer-fw`
3. Add inbound rules for ports 22, 9000, 9443 from your IP
4. Set default inbound policy to **Drop**
5. Assign to your Linode instance

## Step 3: Install Docker

```bash
# SSH to Linode
ssh root@<linode-ip>

# Update system
apt update && apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sh

# Create a non-root user for better security
adduser portaineradmin
usermod -aG docker portaineradmin
usermod -aG sudo portaineradmin

# Switch to the new user
su - portaineradmin
```

## Step 4: Attach Block Storage

```bash
# Via Linode console or CLI, create a 50GB block storage volume
# Attach to your Linode instance

# On the Linode, format and mount
sudo fdisk -l  # Find new volume (usually /dev/disk/by-id/scsi-0Linode_Block_Storage_*)

# Create filesystem
sudo mkfs.ext4 /dev/sdc

# Mount
sudo mkdir -p /mnt/block-storage
echo '/dev/sdc /mnt/block-storage ext4 defaults,noatime,_netdev 0 2' | sudo tee -a /etc/fstab
sudo mount -a

# Use for Docker data
sudo mkdir -p /mnt/block-storage/docker
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "data-root": "/mnt/block-storage/docker"
}
EOF
sudo systemctl restart docker
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

## Step 6: Enable Linode Backups

1. Navigate to your Linode > **Backups**
2. Click **Enable Backups**
3. Set your preferred backup schedule (daily is recommended)

## Step 7: Monitor with Linode's Built-in Metrics

Linode provides basic CPU, network, and disk metrics. For more detailed container metrics, deploy Prometheus and Grafana via Portainer:

```yaml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - prometheus_data:/prometheus
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=secure_password
    volumes:
      - grafana_data:/var/lib/grafana
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    network_mode: host
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

## Conclusion

Linode/Akamai Cloud provides excellent value for Portainer deployments with reliable infrastructure and straightforward pricing. The Cloud Firewall adds a layer of security without OS-level complexity, and block storage volumes support growing container workloads. Linode's built-in backup service provides peace of mind for production deployments.
