# How to Recover Orphaned Stacks in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stack, Recovery, Troubleshooting, DevOps

Description: Learn how to recover orphaned Docker stacks in Portainer - containers running outside Portainer's knowledge that need to be brought back under management.

## Introduction

Orphaned stacks occur when Docker containers are running but Portainer has lost track of them. This happens when Portainer is reinstalled, when containers are deployed directly via CLI without Portainer's knowledge, or when Portainer's database is reset. The containers are healthy and serving traffic, but they don't appear in Portainer's Stacks list. This guide covers how to identify, recover, and re-manage these orphaned deployments.

## Prerequisites

- Portainer installed with a connected Docker environment
- Docker CLI access on the host
- Knowledge of the original Compose file content (if available)

## What Causes Orphaned Stacks

1. **Portainer reinstall/data loss**: Portainer's data volume (`portainer_data`) is deleted or corrupted.
2. **Direct CLI deployment**: `docker compose up` or `docker stack deploy` run directly on the host, bypassing Portainer.
3. **Environment re-addition**: A Docker endpoint is removed from Portainer and re-added - Portainer loses the stack metadata.
4. **Portainer version migration**: Some upgrades don't preserve stack association metadata.

## Step 1: Identify Orphaned Containers

Identify containers that Portainer isn't managing:

```bash
# List all running containers and their Compose project labels:

docker ps --format "table {{.Names}}\t{{.Labels}}" | grep "compose.project"

# Show all containers grouped by Compose project:
docker ps --format '{{.Labels}}' | \
  grep -o 'com.docker.compose.project=[^,]*' | \
  sort -u

# Example output:
# com.docker.compose.project=myapp
# com.docker.compose.project=monitoring
# com.docker.compose.project=traefik
```

Compare this list to what appears in Portainer's Stacks view. Projects present in `docker ps` but missing from Portainer are orphaned.

## Step 2: Reconstruct the Compose File

If you don't have the original Compose file, reconstruct it from running containers:

```bash
# Install docker-autocompose or use manual inspection:
# Option 1: docker-autocompose tool
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  red5d/docker-autocompose \
  myapp_web_1 myapp_api_1 myapp_postgres_1 > reconstructed-stack.yml

# Option 2: Manually inspect each container:
docker inspect myapp_web_1 | jq '.[0] | {
  image: .Config.Image,
  environment: .Config.Env,
  ports: .HostConfig.PortBindings,
  volumes: .Mounts,
  networks: (.NetworkSettings.Networks | keys)
}'
```

## Step 3: Re-Import the Stack into Portainer

### Method 1: Web Editor Import

1. Copy the reconstructed Compose YAML.
2. In Portainer, navigate to **Stacks** → **Add stack**.
3. Enter the **exact same stack name** as the running Compose project (e.g., `myapp`).
4. Select **Web editor** and paste the Compose content.
5. Click **Deploy the stack**.

Portainer detects that containers with this project label already exist and associates them with the new stack record.

### Method 2: Use the Portainer API

```bash
# Get the current Compose file content from container labels:
STACK_NAME="myapp"

# Create the stack entry via API (Portainer will sync with running containers):
curl -X POST \
  "${PORTAINER_URL}/api/stacks" \
  -H "X-API-Key: ${PORTAINER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "'${STACK_NAME}'",
    "SwarmID": "",
    "StackFileContent": "'"$(cat reconstructed-stack.yml | jq -Rs .)"'",
    "Env": [],
    "EndpointId": 1
  }'
```

## Step 4: Handle External Stacks (Deployed via CLI)

Portainer BE has an **External stacks** feature that shows stacks deployed outside Portainer:

1. Navigate to **Stacks** in Portainer.
2. Look for an **External stacks** tab or section.
3. These show stacks that Docker knows about but Portainer didn't deploy.
4. Click **Associate** to bring them under Portainer management.

## Step 5: Prevent Future Orphaning

Strategies to prevent stacks from becoming orphaned:

```bash
# Strategy 1: Always deploy via Portainer (never direct CLI for managed hosts)

# Strategy 2: Backup Portainer data volume regularly:
docker run --rm \
  -v portainer_data:/source:ro \
  -v /backup:/backup \
  alpine tar czf /backup/portainer_data_$(date +%Y%m%d).tar.gz -C /source .

# Strategy 3: Use Git-based stacks - the source of truth is in Git, not Portainer:
# Even if Portainer loses the stack record, you can redeploy from Git
# and the containers will reconnect to existing volumes

# Strategy 4: Export stack definitions regularly:
# Portainer UI: Stacks → (each stack) → copy Compose YAML → save to version control
```

## Step 6: Recover After Portainer Reinstall

If you reinstalled Portainer and lost all stack records:

```bash
# 1. List all running Compose projects:
docker ps --format '{{.Labels}}' | \
  grep -o 'com.docker.compose.project=[^,]*' | \
  sort -u | sed 's/com.docker.compose.project=//'

# 2. For each project, find its compose file (might still be on disk):
find /opt/stacks /home -name "docker-compose.yml" 2>/dev/null

# 3. For each found compose file, redeploy as a stack in Portainer
#    using the same name - containers continue running, Portainer
#    re-associates with them

# 4. Verify each service shows as Running in Portainer after re-association
```

## Conclusion

Orphaned stacks are a common consequence of Portainer data loss or direct CLI deployments. Recovery involves identifying Compose project labels on running containers, reconstructing or locating the original Compose YAML, and deploying a new stack in Portainer with the same project name to re-associate running containers. Going forward, always deploy stacks through Portainer (not directly via CLI), back up Portainer's data volume regularly, and use Git-based stacks so the configuration is never exclusively stored in Portainer's database.
