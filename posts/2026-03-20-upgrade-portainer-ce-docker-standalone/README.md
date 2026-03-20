# How to Upgrade Portainer CE on Docker Standalone

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Upgrade, Maintenance, DevOps

Description: Step-by-step guide to upgrading Portainer Community Edition on a Docker standalone host to the latest version without losing data.

---

Portainer releases updates regularly with bug fixes, security patches, and new features. Upgrading on Docker standalone is a simple pull-and-replace operation that takes less than two minutes.

## Before Upgrading

Always back up your data before upgrading:

```bash
# Back up the Portainer data volume to a local tar archive
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/portainer_backup_$(date +%Y%m%d_%H%M%S).tar.gz -C /data .

echo "Backup created: portainer_backup_$(date +%Y%m%d_%H%M%S).tar.gz"
```

## Step 1: Stop the Current Container

```bash
# Gracefully stop the running Portainer container
docker stop portainer
```

## Step 2: Remove the Container

The data volume (`portainer_data`) is separate from the container and will not be deleted:

```bash
# Remove only the container, not the volume
docker container rm portainer
```

## Step 3: Pull the Latest Image

```bash
# Pull the latest Portainer CE image from Docker Hub
docker pull portainer/portainer-ce:latest
```

## Step 4: Start the Updated Container

Use the same flags as your original installation:

```bash
# Start Portainer CE with the same configuration as before
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 5: Verify the Upgrade

```bash
# Check the container is running with the new image
docker ps --filter name=portainer

# View the exact image version deployed
docker inspect portainer --format '{{.Config.Image}}: {{index .Config.Labels "org.opencontainers.image.version"}}'
```

Open `https://localhost:9443` and check **Settings > About** in the UI to confirm the new version number.

## Upgrading to a Specific Version

To upgrade to a specific version rather than `latest`:

```bash
# Example: upgrade to a specific pinned version
docker pull portainer/portainer-ce:2.21.0

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:2.21.0
```

## Automate with a Script

Save this as `upgrade-portainer.sh` for repeatable upgrades:

```bash
#!/bin/bash
# upgrade-portainer.sh - Upgrade Portainer CE to latest version

set -e

echo "Backing up Portainer data..."
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/portainer_backup_$(date +%Y%m%d_%H%M%S).tar.gz -C /data .

echo "Stopping Portainer..."
docker stop portainer
docker container rm portainer

echo "Pulling latest image..."
docker pull portainer/portainer-ce:latest

echo "Starting updated Portainer..."
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

echo "Upgrade complete!"
docker ps --filter name=portainer
```

---

*Monitor your Docker infrastructure availability with [OneUptime](https://oneuptime.com).*
