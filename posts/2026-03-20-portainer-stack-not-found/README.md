# How to Fix 'Stack Not Found' After a Portainer Crash - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Stack, Recovery

Description: Recover stacks that disappear from Portainer after a crash or restart, including re-importing running stacks and restoring stack definitions from backups.

## Introduction

After Portainer crashes or is reinstalled, you may find that stacks you previously managed are gone from the UI - even though the containers are still running. This happens because Portainer's stack metadata (stored in `portainer.db`) was lost, but the underlying Docker Compose stacks are intact. This guide shows how to recover and re-associate them.

## Understanding the Problem

Portainer stacks consist of:
1. **Container resources**: The actual running containers, networks, and volumes in Docker (persistent, independent of Portainer)
2. **Stack metadata**: Portainer's record of which containers belong to which stack (stored in `portainer.db`)

After a Portainer crash, #2 is lost but #1 survives. The containers continue running but appear as "orphaned" in the Docker view.

## Step 1: Verify Containers Are Still Running

```bash
# Check if the containers are still running

docker ps | grep stack-name

# Check for all containers (including stopped ones)
docker ps -a | grep stack-name

# List containers with their labels (Portainer adds stack labels)
docker ps --format "{{.Names}}: {{.Labels}}" | grep "com.docker.compose.project"
```

## Step 2: Check for Docker Compose Labels

Portainer marks stack containers with Compose labels:

```bash
# Find all stacks by their compose labels
docker ps --format "{{.Names}}" -q | xargs docker inspect \
  --format '{{.Name}}: {{index .Config.Labels "com.docker.compose.project"}}' \
  | grep -v ": $" | sort

# This shows you which containers belong to which stack
```

## Step 3: Re-import Running Stacks into Portainer

Portainer can re-associate existing containers as a stack:

1. In Portainer, go to **Stacks**
2. Click **Add Stack**
3. Choose **Web Editor**
4. Reconstruct the `docker-compose.yml` content (see Step 4)
5. Use the **exact same stack name** as before
6. Check **Deploy** - Portainer will detect running containers and associate them

> **Important**: The stack name must match the Docker Compose project name (`com.docker.compose.project` label).

## Step 4: Reconstruct the Compose File

If you don't have the original compose file:

```bash
# Use docker inspect to reconstruct container configuration
docker inspect stack-name_service-name_1 | jq '.[0]' > /tmp/container-config.json

# Get image
docker inspect --format='{{.Config.Image}}' stack-name_service-name_1

# Get port mappings
docker inspect --format='{{json .HostConfig.PortBindings}}' stack-name_service-name_1

# Get environment variables
docker inspect --format='{{json .Config.Env}}' stack-name_service-name_1

# Get volume mounts
docker inspect --format='{{json .HostConfig.Binds}}' stack-name_service-name_1

# Get networks
docker inspect --format='{{json .NetworkSettings.Networks}}' stack-name_service-name_1
```

Use this information to rebuild the compose file manually.

## Step 5: Use docker-autocompose Tool

```bash
# Install docker-autocompose to auto-generate compose files from running containers
pip3 install docker-autocompose

# Generate compose file from a running stack
docker-autocompose stack-name_service1_1 stack-name_service2_1 > recovered-stack.yml

# Review and clean up the generated file
cat recovered-stack.yml
```

## Step 6: Re-import via Portainer API

```bash
# Authenticate
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Create a new stack pointing to the existing containers
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "http://localhost:9000/api/stacks?type=2&method=string&endpointId=1" \
  -d '{
    "name": "mystack",
    "stackFileContent": "version: \"3.8\"\nservices:\n  myapp:\n    image: nginx:latest\n    ports:\n      - \"80:80\"\n"
  }'
```

## Step 7: Restore from Portainer Backup

If you have a Portainer backup (highly recommended to set up):

```bash
# Stop Portainer
docker stop portainer

# Restore the backup to the data volume
docker run --rm \
  -v portainer_data:/data \
  -v /backup/path:/backup \
  alpine tar xzf /backup/portainer-backup.tar.gz -C /data

# Start Portainer
docker start portainer

# Verify stacks are restored
```

## Step 8: Prevent Future Stack Loss

Set up regular Portainer backups:

```bash
#!/bin/bash
# backup-portainer.sh
BACKUP_DIR="/opt/backups/portainer"
DATE=$(date +%Y%m%d-%H%M%S)

mkdir -p "$BACKUP_DIR"

# Stop Portainer briefly
docker stop portainer

# Backup the data volume
docker run --rm \
  -v portainer_data:/data \
  -v "$BACKUP_DIR":/backup \
  alpine tar czf "/backup/portainer-$DATE.tar.gz" -C /data .

# Restart Portainer
docker start portainer

# Keep only last 7 backups
ls -t "$BACKUP_DIR"/*.tar.gz | tail -n +8 | xargs rm -f

echo "Backup completed: $BACKUP_DIR/portainer-$DATE.tar.gz"
```

Add to crontab: `0 2 * * * /opt/scripts/backup-portainer.sh`

## Step 9: Export Stack Definitions Regularly

```bash
# Use the Portainer API to export all stack definitions
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Get all stacks
STACKS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/stacks)

# Export each stack's compose file
echo "$STACKS" | jq -c '.[]' | while read -r stack; do
  STACK_ID=$(echo "$stack" | jq -r '.Id')
  STACK_NAME=$(echo "$stack" | jq -r '.Name')

  curl -s -H "Authorization: Bearer $TOKEN" \
    "http://localhost:9000/api/stacks/$STACK_ID/file" | \
    jq -r '.StackFileContent' > "/opt/stacks-backup/$STACK_NAME.yml"

  echo "Exported: $STACK_NAME"
done
```

## Conclusion

"Stack Not Found" after a Portainer crash doesn't mean your containers are gone - they're likely still running fine. Recover by re-importing them using their existing Docker Compose definitions, or by reconstructing the compose files from `docker inspect` output. The best long-term solution is regular automated backups of the Portainer data volume and periodic exports of stack definitions to a version-controlled directory.
