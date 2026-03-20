# How to Fix Duplicate Resources Appearing with Agent Endpoints - Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Agent, Resources

Description: Fix the issue where Portainer shows duplicate containers, networks, or volumes when using agent-based endpoints, caused by multiple agent registrations or snapshot conflicts.

## Introduction

If you're seeing the same container listed twice in Portainer, or networks and volumes appearing multiple times, this is typically caused by the Portainer server having multiple endpoint registrations pointing to the same Docker host, or by agent cluster mode creating duplicate entries. This guide explains how to diagnose and resolve the issue.

## Step 1: Check for Duplicate Environments

```bash
# List all environments via Portainer API

TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints | \
  jq '.[] | {id: .Id, name: .Name, url: .URL, type: .Type}'
```

Look for two entries with the same URL or pointing to the same Docker host. This is the most common cause of duplicate resources.

## Step 2: Remove Duplicate Environments

1. In Portainer UI, go to **Environments**
2. Identify the duplicate entries (same URL, same agent)
3. Remove the extra one:
   - Click the environment → **Remove**
   - Confirm deletion

Only the Portainer configuration is removed - the Docker host and its containers are unaffected.

## Step 3: Check for Multiple Agent Instances

```bash
# On the Docker host, check how many agent containers are running
docker ps | grep portainer-agent

# There should be EXACTLY ONE agent per Docker host
# If you see multiple:
docker stop portainer-agent portainer-agent2  # stop extras
docker rm portainer-agent2  # remove the extra

# Keep only one agent running
docker restart portainer-agent
```

## Step 4: Fix Swarm Agent Cluster Duplicates

In Docker Swarm, the agent uses a cluster mode that aggregates all nodes. If you've also added each individual Swarm node as a separate endpoint, you'll see duplicates:

```bash
# In Portainer, you should have ONE of these, not both:
# Option A: Swarm endpoint (connects to the agent service, sees all nodes)
# Option B: Direct endpoints for each individual node

# Remove individual node endpoints if you have a Swarm endpoint
# Or remove the Swarm endpoint if you want per-node management
```

The cluster endpoint shows the aggregate Swarm view; individual endpoints show per-node views. Choose one approach.

## Step 5: Check AGENT_CLUSTER_ADDR Configuration

Incorrect cluster configuration can cause duplicate reporting:

```bash
# Check the agent's AGENT_CLUSTER_ADDR setting
docker inspect portainer-agent | grep AGENT_CLUSTER_ADDR

# If running in Swarm cluster mode and seeing duplicates:
# Ensure AGENT_CLUSTER_ADDR is set to the service DNS name
# AGENT_CLUSTER_ADDR=tasks.portainer-agent

# If running standalone (single host), remove the AGENT_CLUSTER_ADDR
# to prevent cluster formation behavior:
docker stop portainer-agent && docker rm portainer-agent

docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
  # Note: No AGENT_CLUSTER_ADDR for standalone mode
```

## Step 6: Force a Fresh Snapshot

Sometimes duplicates are a display artifact from a stale snapshot:

```bash
# Trigger a fresh snapshot via API
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Get all endpoint IDs
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints | jq '.[].Id'

# Trigger snapshot for endpoint 1
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints/1/docker/snapshot

# Refresh the Portainer UI after the snapshot completes
```

## Step 7: Check for Stale Endpoint Data

If an old endpoint was removed but its data persists:

```bash
# Check Portainer database for stale endpoint references
# This requires database inspection
docker run --rm \
  -v portainer_data:/data \
  alpine ls -la /data/

# The portainer.db contains all endpoint data
# Deleting it resets all Portainer configuration (use as last resort)
```

## Step 8: Clear Browser Cache

Sometimes duplicate display is a browser rendering artifact:

```bash
# Hard reload the page
# Chrome/Firefox: Ctrl+Shift+R (Windows/Linux) or Cmd+Shift+R (Mac)

# Or clear local storage for the Portainer origin
# DevTools → Application → Local Storage → Clear
```

## Step 9: Check Volume and Network Ownership

Docker volumes and networks have labels indicating which container created them. If labels are missing or duplicated:

```bash
# Check volume labels
docker volume ls --format "{{.Name}}\t{{.Labels}}" | grep -v "^$"

# Check network labels
docker network ls --format "{{.Name}}\t{{.Labels}}"

# Volumes and networks without proper labels may appear multiple times
# in Portainer's inventory
```

## Conclusion

Duplicate resources in Portainer almost always stem from having multiple environment registrations pointing to the same Docker host, or from running multiple agent instances on a single host. Start by checking for duplicate environments in the Portainer UI, then verify only one agent is running per Docker host. Force a snapshot refresh to clear any stale display artifacts.
