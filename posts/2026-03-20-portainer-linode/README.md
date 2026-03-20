# How to Deploy Portainer on Linode/Akamai Cloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Linode, Akamai Cloud, Docker, Cloud Deployment

Description: Learn how to deploy Portainer on a Linode (now Akamai Cloud) instance with StackScripts for automated setup, firewall configuration, and reverse DNS setup.

## Prerequisites

- Linode/Akamai Cloud account
- SSH key added to your account

## Option 1: Deploy via Linode Cloud Manager

1. Go to **Linodes → Create Linode**
2. Select **Distributions → Ubuntu 22.04 LTS**
3. Choose plan: **Nanode 1GB** (minimum) or **Linode 2GB** (recommended)
4. Select region
5. Under **Advanced Options → Add a StackScript** or use **User Data**:

```bash
#!/bin/bash
# Install Docker

curl -fsSL https://get.docker.com | sh
usermod -aG docker root

# Install Portainer
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

## Option 2: Deploy via Linode CLI

```bash
# Install and configure linode-cli
pip install linode-cli
linode-cli configure

# Create the instance
linode-cli linodes create \
  --type g6-standard-1 \
  --region us-east \
  --image linode/ubuntu22.04 \
  --label portainer-server \
  --root_pass "$(openssl rand -base64 24)" \
  --authorized_keys "$(cat ~/.ssh/id_rsa.pub)"

# Get IP
linode-cli linodes list --format "label,ipv4" --text | grep portainer
```

## Step 3: Install Portainer Manually

```bash
# SSH into your Linode
ssh root@<LINODE_IP>

# Install Docker
curl -fsSL https://get.docker.com | sh

# Create non-root user (recommended)
adduser deploy
usermod -aG docker deploy
usermod -aG sudo deploy

# Install Portainer as deploy user
su - deploy

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

## Configure Linode Firewall

```bash
# Create firewall via API or use Cloud Manager
linode-cli firewalls create \
  --label portainer-fw \
  --rules.inbound '[
    {"action":"ACCEPT","protocol":"TCP","ports":"22"},
    {"action":"ACCEPT","protocol":"TCP","ports":"9443"},
    {"action":"ACCEPT","protocol":"TCP","ports":"80,443"},
    {"action":"DROP","protocol":"TCP","ports":"1-65535"}
  ]' \
  --rules.outbound_policy ACCEPT \
  --rules.inbound_policy DROP

# Attach to Linode
linode-cli firewalls devices-create <FIREWALL_ID> \
  --type linode \
  --id <LINODE_ID>
```

## Set Reverse DNS

Linode allows setting rDNS for your IP:

```bash
linode-cli linodes ip-update <LINODE_ID> <LINODE_IP> \
  --rdns portainer.yourdomain.com
```

## Enable Linode Backups

```bash
# Enable automated backups ($2/month for Nanode)
linode-cli linodes backups-enable <LINODE_ID>
```

## Recommended Plans for Portainer

| Plan | RAM | vCPU | Storage | Price |
|------|-----|------|---------|-------|
| Nanode 1GB | 1GB | 1 | 25GB SSD | $5/mo |
| Linode 2GB | 2GB | 1 | 50GB SSD | $12/mo |
| Linode 4GB | 4GB | 2 | 80GB SSD | $24/mo |

The 2GB plan provides a comfortable buffer for Portainer plus a few services.

## Conclusion

Linode (Akamai Cloud) provides reliable, predictable-cost VMs for running Portainer. The StackScript/User Data approach automates the entire setup so you get a running Portainer instance within minutes of Linode creation. Linode's firewall and backup features round out a production-ready deployment.
