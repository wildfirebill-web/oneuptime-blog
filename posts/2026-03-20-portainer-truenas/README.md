# How to Install Portainer on TrueNAS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, TrueNAS, NAS, Docker, Self-Hosted, Home Lab

Description: Install Portainer on TrueNAS SCALE to manage Docker containers and Kubernetes applications through a unified web interface.

## Introduction

TrueNAS SCALE includes its own Apps system built on Kubernetes (k3s), but some users prefer Docker-only workflows managed through Portainer. TrueNAS SCALE also supports Docker natively alongside its Apps system. This guide covers installing Portainer on both TrueNAS SCALE and TrueNAS CORE.

## Prerequisites

- TrueNAS SCALE 22.12 or later (for Docker support) OR TrueNAS CORE with a jail/VM
- SSH access enabled
- At least 4GB RAM

## TrueNAS SCALE: Docker Method

### Step 1: Enable Docker on TrueNAS SCALE

TrueNAS SCALE 24.10+ supports Docker natively. Enable it:

1. Navigate to **Apps > Settings**
2. Under **Container Runtime**, select **Docker**
3. Click **Save**

### Step 2: SSH and Deploy Portainer

```bash
# SSH into TrueNAS SCALE

ssh admin@<truenas-ip>

# Create a dataset for Portainer data (recommended)
# Use the TrueNAS UI: Storage > Create Dataset: apps/portainer

# Create the Portainer volume pointing to the dataset
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/mnt/pool/apps/portainer \
  --opt o=bind \
  portainer_data

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

### Step 3: Compose-Based Deployment

Create a Docker Compose file on TrueNAS SCALE:

```bash
# Create directory for compose file
mkdir -p /mnt/pool/apps/portainer-compose

cat > /mnt/pool/apps/portainer-compose/docker-compose.yml << 'EOF'
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      # Docker socket for container management
      - /var/run/docker.sock:/var/run/docker.sock
      # Store data on TrueNAS dataset
      - /mnt/pool/apps/portainer:/data
EOF

cd /mnt/pool/apps/portainer-compose
docker-compose up -d
```

## TrueNAS SCALE: App Catalog Method

If using the built-in App system (Kubernetes-based):

1. Navigate to **Apps > Discover Apps**
2. Search for `Portainer`
3. If available in the catalog, click **Install**
4. Configure the chart values:
   - Service type: NodePort or LoadBalancer
   - Port: 9000
   - Persistence: enable with your preferred storage class

## TrueNAS CORE: Using a Jail

TrueNAS CORE uses FreeBSD jails, not Linux containers. Install Docker in a Linux VM instead:

### Option A: Linux VM

1. Navigate to **Virtualization > Add Virtual Machine**
2. Create an Ubuntu 22.04 LTS VM
3. Allocate 2 vCPUs, 4GB RAM, 20GB disk
4. Install Docker inside the VM
5. Deploy Portainer inside the VM

```bash
# Inside the Ubuntu VM
# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Deploy Portainer
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

## Step 4: Configure TrueNAS Firewall

Allow Portainer ports through TrueNAS firewall:

1. Navigate to **Network > Firewall** (if enabled)
2. Add rules for TCP ports 9000 and 9443
3. Restrict source to your local subnet

## Step 5: Access Portainer

Navigate to `http://<truenas-ip>:9000` and create your admin account.

## Backing Up Portainer Data

TrueNAS datasets support ZFS snapshots. Schedule snapshots for your Portainer data:

1. Navigate to **Data Protection > Periodic Snapshot Tasks**
2. Add a task for the `apps/portainer` dataset
3. Set schedule (daily recommended)
4. Set retention (7 days)

## Conclusion

Portainer on TrueNAS gives you a powerful container management interface backed by ZFS storage reliability. Using a TrueNAS dataset for Portainer data means you benefit from ZFS checksums, snapshots, and replication for your container configuration. TrueNAS SCALE's Docker support makes this particularly straightforward in recent releases.
