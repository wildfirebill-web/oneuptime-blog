# How to Install Portainer on QNAP NAS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, QNAP, NAS, Docker, Container Station, Self-Hosted, Home Lab

Description: Install Portainer on a QNAP NAS using Container Station to get a powerful Docker management interface beyond what QNAP's native tools provide.

## Introduction

QNAP's Container Station provides basic Docker management, but it lacks many advanced features. Installing Portainer on your QNAP NAS gives you full stack support, environment templates, registry management, and a far more capable interface for managing complex containerized applications.

## Prerequisites

- QNAP NAS running QTS 5.x or QuTS hero
- Container Station 3.x installed from App Center
- At least 2GB RAM available
- SSH access enabled (Control Panel > Network & Virtual Switch > Services > SSH)

## Method 1: Via Container Station UI

### Step 1: Pull the Portainer Image

1. Open **Container Station**
2. Click **Images** in the left sidebar
3. Click **Pull**
4. Enter `portainer/portainer-ce` and tag `latest`
5. Click **Pull**

### Step 2: Create the Container

1. Click **Containers > Create**
2. Select the `portainer/portainer-ce:latest` image
3. Click **Advanced Settings**

Configure the following:

**Network:** Select `Host` or create a bridge mapping:
- Host port `9000` → Container port `9000`
- Host port `9443` → Container port `9443`

**Storage:**
- Click **Add Volume**
- Type: Docker Volume, Name: `portainer_data`, Mount path: `/data`
- Click **Add Bind Mount**
- Host path: `/var/run/docker.sock`, Mount path: `/var/run/docker.sock`
- Access mode: Read/Write

**Auto Restart:** Enable

4. Click **Create**

## Method 2: Via SSH (Recommended)

SSH into your QNAP NAS:

```bash
ssh admin@<qnap-ip>
```

Install Portainer via Docker CLI:

```bash
# Create persistent data volume

docker volume create portainer_data

# Run Portainer
docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Check it's running
docker ps | grep portainer
```

## Method 3: Via Container Station Compose (QTS 5.1+)

Container Station 3.x supports Docker Compose. Create a new application:

1. Open **Container Station**
2. Click **Applications > Create**
3. Enter application name: `portainer`
4. Paste this compose file:

```yaml
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
      # Mount Docker socket for container management
      - /var/run/docker.sock:/var/run/docker.sock
      # Persistent data volume
      - portainer_data:/data

volumes:
  portainer_data:
    driver: local
```

5. Click **Create**

## Step 3: Configure QNAP Firewall

If QNAP's network firewall is enabled:

1. Go to **Control Panel > Security > Security Level**
2. Click **Edit Rules**
3. Add rules to allow TCP ports `9000` and `9443` from your local subnet
4. Ensure the allow rules are ordered before any deny rules

## Step 4: Access Portainer

Navigate to `http://<qnap-ip>:9000` in your browser. Create your admin account on first access.

## Troubleshooting QNAP-Specific Issues

### Docker Socket Permission Denied

QNAP may have different socket permissions:

```bash
# Check socket permissions
ls -la /var/run/docker.sock

# Fix if needed
chmod 666 /var/run/docker.sock
```

### Container Station Conflict

If Container Station manages the same containers:

```bash
# List all containers including Container Station ones
docker ps -a

# Check if Container Station is interfering
docker info | grep -i "docker root"
```

### Port Already in Use

If 9000 is already used by QNAP services:

```bash
# Check which process uses port 9000
netstat -tlnp | grep 9000

# Use an alternative port
docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 19000:9000 \    # Map to 19000 instead
  -p 19443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Updating Portainer on QNAP

```bash
# SSH into QNAP
ssh admin@<qnap-ip>

# Stop and remove old container
docker stop portainer && docker rm portainer

# Pull latest image
docker pull portainer/portainer-ce:latest

# Recreate (data volume is preserved)
docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Conclusion

Portainer on QNAP NAS unlocks full Docker stack management capabilities that Container Station doesn't provide. Whether you use the UI, SSH, or the Container Station Compose feature, the result is a powerful management interface that complements QNAP's native tools. With persistent volumes, your Portainer configuration survives updates and reboots.
