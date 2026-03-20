# How to Install Portainer on ASUSTOR NAS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, ASUSTOR, NAS, Docker, Self-Hosted, Home Lab

Description: Install Portainer on an ASUSTOR NAS to manage Docker containers with a full-featured web interface beyond what ASUSTOR's native Docker app provides.

## Introduction

ASUSTOR NAS devices running ADM 4.x support Docker through the **Docker** app in App Central. Like other NAS vendors, ASUSTOR's Docker UI is limited. Installing Portainer gives you stack management, environment templates, and a more capable container management experience.

## Prerequisites

- ASUSTOR NAS running ADM 4.2 or later
- Docker app installed from App Central
- SSH access enabled (Settings > Services > SSH Terminal)
- At least 2GB RAM

## Step 1: Enable Docker on ASUSTOR

1. Open **App Central**
2. Search for and install **Docker**
3. Open the Docker app to initialize it
4. Note the Docker data location (typically `/volume1/.docker`)

## Step 2: SSH into the NAS

```bash
ssh admin@<asustor-ip>
```

## Step 3: Install Portainer via CLI

```bash
# Create a directory on the NAS volume for Portainer data
mkdir -p /volume1/Docker/portainer

# Create a Docker volume pointing to NAS storage
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/volume1/Docker/portainer \
  --opt o=bind \
  portainer_data

# Run Portainer
docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Verify
docker ps | grep portainer
```

## Step 4: Use Docker Compose

ASUSTOR supports Docker Compose. Create a compose file:

```bash
# Create the compose directory
mkdir -p /volume1/Docker/portainer-compose

cat > /volume1/Docker/portainer-compose/docker-compose.yml << 'EOF'
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
      # Persistent storage on NAS volume
      - /volume1/Docker/portainer:/data

EOF

# Deploy
cd /volume1/Docker/portainer-compose
docker-compose up -d
```

## Step 5: Configure ADM Firewall

1. Open **ADM > Settings > Security > Firewall**
2. Enable the firewall if not already enabled
3. Click **Edit Rules**
4. Add rules:
   - Protocol: TCP, Port: 9000, Action: Allow, Source: Local subnet
   - Protocol: TCP, Port: 9443, Action: Allow, Source: Local subnet
5. Move allow rules above any default deny rules

## Step 6: Access Portainer

Navigate to `http://<asustor-ip>:9000` and complete the initial setup wizard.

## Step 7: Configure Auto-Start via ADM

ASUSTOR doesn't have a Task Scheduler like Synology, but Docker containers with `--restart=unless-stopped` will restart automatically when the Docker service starts after reboot.

To verify:

```bash
# Check restart policy is set
docker inspect portainer | grep -A3 RestartPolicy
```

Output should show:
```json
"RestartPolicy": {
    "Name": "unless-stopped",
    "MaximumRetryCount": 0
}
```

## Troubleshooting ASUSTOR-Specific Issues

### Docker Socket Permission Issues

```bash
# Check socket permissions
ls -la /var/run/docker.sock

# Add admin user to docker group if needed
sudo usermod -aG docker admin
```

### Container Not Starting After Reboot

ASUSTOR Docker may not start automatically at boot. Enable it:

1. Open **App Central > Docker**
2. Enable **Auto-start** in Docker settings

Or create a startup script:

```bash
# Create startup script
cat > /usr/local/etc/rc.d/portainer-start.sh << 'EOF'
#!/bin/sh
sleep 30  # Wait for Docker to start
docker start portainer
EOF

chmod +x /usr/local/etc/rc.d/portainer-start.sh
```

## Updating Portainer

```bash
ssh admin@<asustor-ip>

# Update Portainer
docker stop portainer && docker rm portainer
docker pull portainer/portainer-ce:latest
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

Portainer transforms your ASUSTOR NAS into a capable container management platform. By storing data on the NAS volume, your Portainer configuration is part of your existing backup strategy. The Docker restart policy ensures Portainer comes back up automatically after reboots or power failures.
