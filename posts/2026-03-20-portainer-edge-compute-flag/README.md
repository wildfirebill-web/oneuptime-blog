# How to Use the --edge-compute Flag to Enable Edge Features

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Edge Computing, CLI, Configuration

Description: Enable Portainer's Edge Computing features using the --edge-compute flag, allowing management of remote and air-gapped Docker environments via reverse tunnel connections.

## Introduction

Portainer's Edge Computing capabilities allow you to manage Docker environments that are not directly reachable from the Portainer server - remote locations, air-gapped networks, and IoT devices. The `--edge-compute` flag enables these features when starting the Portainer server.

## What --edge-compute Enables

When `--edge-compute` is set:
- The Edge Agent tunnel endpoint (port 8000) becomes active
- Edge environment management UI becomes available
- Edge groups and edge stacks are unlocked
- Remote job scheduling for edge devices is enabled
- Async polling mode for disconnected environments

## Step 1: Enable Edge Compute in Portainer

```bash
# Start Portainer with Edge Compute enabled

docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  -p 8000:8000 \         # Edge Agent tunnel port - REQUIRED
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --edge-compute           # Enable edge features

# Access: http://your-host:9000
# Edge Compute section now visible in the sidebar
```

## Step 2: Verify Edge Compute Is Enabled

```bash
# Check Portainer logs for edge compute initialization
docker logs portainer 2>&1 | grep -i "edge\|tunnel" | head -10

# Verify port 8000 is listening
ss -tlnp | grep 8000

# Via API
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/settings | jq '.EdgeAgentCheckinInterval'
```

## Step 3: Create an Edge Environment

1. In Portainer, go to **Environments** → **Add Environment**
2. Select **Docker Standalone** or **Docker Swarm**
3. Choose **Edge Agent**
4. Name the environment (e.g., "remote-site-1")
5. Configure:
   - **Portainer server URL**: `https://portainer.yourdomain.com`
   - **Portainer tunnel server**: `portainer.yourdomain.com:8000`
6. Portainer generates a deployment command

## Step 4: Deploy Edge Agent on Remote Device

Copy the generated command to the remote device:

```bash
# Generated command looks like:
docker run -d \
  --name portainer-edge-agent \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /:/host \
  -v portainer_agent_data:/data \
  -e EDGE=1 \
  -e EDGE_ID=<generated-uuid> \
  -e EDGE_KEY=<generated-key> \
  -e EDGE_INSECURE_POLL=1 \
  portainer/agent:latest
```

## Step 5: Configure Edge Agent Check-in Interval

```bash
# The edge agent polls the server periodically
# Configure check-in interval via Portainer settings

# Via API (set to 30 seconds for example)
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:9000/api/settings \
  -d '{"EdgeAgentCheckinInterval": 30}'

# Via UI: Settings → Edge Compute → Edge Agent Check-in interval
```

## Step 6: Create Edge Groups

Edge groups allow you to manage multiple edge devices together:

1. In Portainer, go to **Edge Compute** → **Edge Groups**
2. Click **Add Edge Group**
3. Choose **Static** (manual assignment) or **Dynamic** (tag-based)
4. For static: add environments manually
5. For dynamic: configure tags that match environments

```bash
# Create an edge group via API
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:9000/api/edge_groups \
  -d '{
    "name": "production-sites",
    "dynamic": false,
    "endpoints": [2, 3, 4]
  }'
```

## Step 7: Deploy an Edge Stack

Edge stacks deploy compose files to multiple edge environments:

1. Go to **Edge Compute** → **Edge Stacks**
2. Click **Add Edge Stack**
3. Select the Edge Groups to deploy to
4. Provide the compose file:

```yaml
# Example edge stack
version: "3.8"
services:
  data-collector:
    image: mycompany/data-collector:v1.2
    restart: unless-stopped
    environment:
      - COLLECTION_INTERVAL=60s
      - UPLOAD_ENDPOINT=https://api.mycompany.com
    volumes:
      - collector_data:/data

volumes:
  collector_data:
```

## Step 8: Configure Edge Async Mode

For environments with poor connectivity:

```bash
# Configure async mode for specific edge environments
# Go to: Environments → select edge environment → Advanced settings
# Enable: Use asynchronous mode for this environment

# In async mode:
# - Portainer sends a snapshot request
# - Agent processes when it reconnects
# - No real-time connection required
```

## Step 9: Run Remote Jobs on Edge Devices

```bash
# Schedule a script to run on edge devices
# Via UI: Edge Compute → Edge Jobs → Add Edge Job

# Via API
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:9000/api/edge_jobs \
  -d '{
    "Name": "health-check",
    "CronExpression": "0 */6 * * *",
    "Endpoints": [2, 3],
    "ScriptPath": "/opt/scripts/health-check.sh",
    "FileContent": "#!/bin/bash\ndocker ps\ndf -h"
  }'
```

## Step 10: Docker Compose for Edge-Enabled Portainer

```yaml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    command: >
      --edge-compute
      --tunnel-addr=0.0.0.0
      --tunnel-port=8000
    ports:
      - "9000:9000"
      - "9443:9443"
      - "8000:8000"   # MUST expose for Edge Agents to connect
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

## Conclusion

The `--edge-compute` flag unlocks Portainer's ability to manage remote and air-gapped Docker environments through an outbound-only reverse tunnel. Always pair it with `-p 8000:8000` to expose the tunnel port, and ensure port 8000 is reachable from your edge devices. Edge compute transforms Portainer from a local container manager into a centralized management platform for distributed infrastructure.
