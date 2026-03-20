# How to Upgrade Portainer CE on Docker Standalone

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, upgrade, docker-standalone, update

Description: A step-by-step guide to upgrading Portainer CE on a Docker standalone host, ensuring zero data loss and minimal downtime.

## Overview

Upgrading Portainer CE on a Docker standalone host is a straightforward process involving pulling the new image, stopping the old container, and starting a new one using the same data volume. This guide covers the standard upgrade process and best practices for production upgrades.

## Prerequisites

- Portainer CE installed on Docker standalone
- Docker CLI access to the host
- Backup of Portainer data (recommended)

## Step 1: Backup Before Upgrading

Always back up before upgrading:

```bash
# Backup Portainer data volume
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/portainer-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /data .

echo "Backup created: portainer-backup-$(date +%Y%m%d).tar.gz"
```

## Step 2: Check Current Version

```bash
# Check current Portainer version
docker exec portainer /portainer --version
# or
docker image inspect portainer | grep -i version
```

## Step 3: Pull New Image

```bash
# Pull the latest Portainer CE image
docker pull portainer/portainer-ce:latest

# Or pull a specific version
docker pull portainer/portainer-ce:2.20.2
```

## Step 4: Stop and Remove Old Container

```bash
# Stop Portainer
docker stop portainer

# Remove the old container (data volume is NOT removed)
docker rm portainer
```

## Step 5: Start New Container

```bash
# Start new Portainer container with the same volume
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 6: Verify Upgrade

```bash
# Check container is running
docker ps | grep portainer

# Check logs for startup errors
docker logs portainer

# Check version in UI
# Portainer UI -> Help -> About -> Current version
```

## Step 7: Clean Up Old Images

```bash
# Remove old Portainer images to free disk space
docker image prune -f

# Or remove specific old version
docker rmi portainer/portainer-ce:2.19.0
```

## Automated Upgrade Script

```bash
#!/bin/bash
# upgrade-portainer.sh

set -e

echo "Starting Portainer upgrade..."

# Backup
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/portainer-pre-upgrade-$(date +%Y%m%d-%H%M%S).tar.gz -C /data .
echo "Backup created"

# Pull new image
docker pull portainer/portainer-ce:latest
echo "New image pulled"

# Get current container configuration
PORTS=$(docker inspect portainer --format '{{range $p, $conf := .HostConfig.PortBindings}}{{(index $conf 0).HostPort}}:{{$p}} {{end}}' 2>/dev/null || echo "8000:8000 9443:9443")
MOUNTS=$(docker inspect portainer --format '{{range .Mounts}}-v {{.Source}}:{{.Destination}} {{end}}' 2>/dev/null)

# Stop and remove old container
docker stop portainer
docker rm portainer
echo "Old container removed"

# Start new container
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

echo "Portainer upgraded successfully!"
docker ps | grep portainer
```

## Downtime Consideration

The upgrade process involves a brief period (seconds) where Portainer is unavailable while the old container is stopped and the new one starts. This is typically acceptable for maintenance windows.

For zero-downtime upgrades in team environments, consider:
1. Upgrading during off-hours
2. Notifying users before the maintenance window
3. Having the upgrade script ready to execute quickly

## Conclusion

Upgrading Portainer CE on Docker standalone is a simple, low-risk operation. The key insight is that the data volume (`portainer_data`) persists independently of the container, so replacing the container with a new image version preserves all configuration. Always backup before upgrading and verify the upgrade was successful by checking logs and the UI version display.
