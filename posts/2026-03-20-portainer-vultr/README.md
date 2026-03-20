# How to Deploy Portainer on Vultr Cloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Vultr, Docker, Cloud, VPS

Description: Learn how to deploy Portainer on a Vultr Cloud Compute instance with startup scripts, firewall rules, and Vultr's block storage for persistent data.

## Prerequisites

- Vultr account
- Vultr CLI or API access
- SSH key uploaded to Vultr

## Option 1: Deploy via Vultr Control Panel

1. **Products → Compute → Deploy Server**
2. Cloud Compute → Regular Performance
3. Location: choose nearest region
4. Image: Ubuntu 22.04 LTS
5. Size: 2 vCPU / 4GB RAM (~$24/mo) or 1 vCPU / 2GB RAM (~$12/mo)
6. Under **Server Settings → Startup Script**:

```bash
#!/bin/bash
apt-get update
curl -fsSL https://get.docker.com | sh
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

## Option 2: Deploy via Vultr CLI

```bash
# Install vultr-cli
go install github.com/vultr/vultr-cli/v2@latest

# Configure API key
export VULTR_API_KEY="your-api-key"

# Create instance
vultr-cli instance create \
  --region ewr \
  --plan vc2-2c-4gb \
  --os 1743 \            # Ubuntu 22.04 LTS OS ID
  --label portainer-server \
  --ssh-keys YOUR_KEY_ID \
  --notify-activate true

# List instances and get IP
vultr-cli instance list --output=json | jq '.instances[] | {label, main_ip}'
```

## Configure Vultr Firewall

```bash
# Create firewall group
FIREWALL_ID=$(vultr-cli firewall group create \
  --description "Portainer Firewall" | grep -oP '"id":"\K[^"]+')

# Add inbound rules
vultr-cli firewall rule create \
  --firewall-group-id $FIREWALL_ID \
  --ip-type v4 \
  --protocol tcp \
  --port 22 \
  --size 0 \
  --subnet "0.0.0.0" \
  --notes "SSH"

vultr-cli firewall rule create \
  --firewall-group-id $FIREWALL_ID \
  --ip-type v4 \
  --protocol tcp \
  --port 9443 \
  --size 0 \
  --subnet "0.0.0.0" \
  --notes "Portainer HTTPS"

vultr-cli firewall rule create \
  --firewall-group-id $FIREWALL_ID \
  --ip-type v4 \
  --protocol tcp \
  --port 443 \
  --size 0 \
  --subnet "0.0.0.0" \
  --notes "HTTPS"

# Attach firewall to instance
vultr-cli instance update INSTANCE_ID --firewall-group-id $FIREWALL_ID
```

## Attach Block Storage

For persistent data that survives instance replacement:

```bash
# Create block storage volume
VOLUME_ID=$(vultr-cli block-storage create \
  --region ewr \
  --size 20 \
  --label portainer-storage | grep -oP '"id":"\K[^"]+')

# Attach to instance
vultr-cli block-storage attach \
  --block-storage-id $VOLUME_ID \
  --instance-id INSTANCE_ID \
  --live

# On the server: format and mount
mkfs.ext4 /dev/vdb
mkdir -p /mnt/portainer-data
mount /dev/vdb /mnt/portainer-data
echo "/dev/vdb /mnt/portainer-data ext4 defaults 0 0" >> /etc/fstab
```

## Set Up Vultr DNS

```bash
# Create DNS record via API
curl -s "https://api.vultr.com/v2/domains" \
  -X POST \
  -H "Authorization: Bearer $VULTR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "yourdomain.com"}'

# Add A record for Portainer
curl -s "https://api.vultr.com/v2/domains/yourdomain.com/records" \
  -X POST \
  -H "Authorization: Bearer $VULTR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"type": "A", "name": "portainer", "data": "YOUR_SERVER_IP", "ttl": 300}'
```

## Conclusion

Vultr's compute instances are straightforward for Portainer deployments. The startup script feature automates Docker and Portainer installation, and Vultr's block storage provides persistent, detachable storage for Portainer data — useful when migrating between instance types or regions.
