# How to Set Up Cloudflare Tunnel for Portainer Edge Agents - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Cloudflare, Edge Agent, Zero Trust, Remote Management

Description: Learn how to use Cloudflare Tunnel to connect Portainer Edge Agents running in remote or private networks to your Portainer server, enabling secure management without VPN or direct port exposure.

## Introduction

Portainer Edge Agents allow Portainer to manage Docker environments in remote networks. Traditionally, Edge Agents need to reach the Portainer server over the internet. Cloudflare Tunnel provides a secure relay for this connection, allowing Edge Agents in locked-down environments to communicate with your Portainer server without opening inbound firewall ports on either side.

## Prerequisites

- Portainer Business Edition or CE with Edge Agent support
- Cloudflare Zero Trust account
- Two environments: Portainer server and remote host (the edge)
- cloudflared available on both sides

## Step 1: Architecture Overview

```text
Remote Host (Edge)                    Your Network
┌──────────────┐                    ┌──────────────────┐
│ Edge Agent   │──cloudflared──▶CF◀──cloudflared──│ Portainer Server │
│ + cloudflared│     Tunnel              Tunnel   │ (Central)        │
└──────────────┘                    └──────────────────┘

Flow:
1. Portainer server exposed via Cloudflare Tunnel
2. Edge Agent connects to Portainer via tunnel URL
3. No inbound firewall ports needed on either side
```

## Step 2: Expose Portainer Server via Cloudflare Tunnel

On your Portainer server, set up a tunnel exposing the Edge port:

```yaml
# /opt/cloudflared/config.yml on Portainer server

tunnel: YOUR_TUNNEL_UUID
credentials-file: /etc/cloudflared/YOUR_TUNNEL_UUID.json

ingress:
  # Standard Portainer UI
  - hostname: portainer.example.com
    service: http://portainer:9000

  # Edge Agent communication port (8000)
  - hostname: edge.portainer.example.com
    service: http://portainer:8000

  - service: http_status:404
```

```bash
# Create DNS routes for edge communication
cloudflared tunnel route dns portainer-tunnel edge.portainer.example.com
```

## Step 3: Configure Portainer for Edge Agents

In Portainer settings, configure the Edge URL:

1. Go to **Settings** → **Edge Compute**
2. Set **Portainer URL**: `https://edge.portainer.example.com`
3. This URL is what Edge Agents will connect to

## Step 4: Create an Edge Environment in Portainer

1. Go to **Environments** → **Add environment**
2. Select **Docker Standalone** → **Edge Agent**
3. Portainer generates an Edge ID and Edge Key
4. Copy the generated docker run command for the remote host

The command looks like:
```bash
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /:/host \
  -v portainer_agent_data:/data \
  --restart always \
  -e EDGE=1 \
  -e EDGE_ID=your-edge-id \
  -e EDGE_KEY=your-edge-key \
  -e EDGE_INSECURE_POLL=1 \    # Remove if using trusted cert
  --name portainer_edge_agent \
  portainer/agent:latest
```

## Step 5: Deploy Edge Agent Behind Cloudflare Tunnel (Remote Side)

On the remote host that the Edge Agent will manage:

```yaml
# docker-compose.yml on remote host
version: "3.8"

services:
  edge-agent:
    image: portainer/agent:latest
    restart: always
    environment:
      EDGE: "1"
      EDGE_ID: "${EDGE_ID}"          # From Portainer environment setup
      EDGE_KEY: "${EDGE_KEY}"        # From Portainer environment setup
      EDGE_INSECURE_POLL: "0"        # 0 = verify TLS (recommended with Cloudflare)
      AGENT_CLUSTER_ADDR: ""         # Not needed for edge agent
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
      - /:/host
      - edge_agent_data:/data

volumes:
  edge_agent_data:
```

The Edge Agent connects out to `edge.portainer.example.com` (your Portainer server via Cloudflare Tunnel).

## Step 6: Optional - Cloudflare Tunnel on Remote Host

If the remote host also needs to be reached for other purposes (or for a more complex setup):

```yaml
# docker-compose.yml on remote host with cloudflared
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command: tunnel --no-autoupdate run
    environment:
      TUNNEL_TOKEN: "${REMOTE_TUNNEL_TOKEN}"    # Different tunnel for remote host

  edge-agent:
    image: portainer/agent:latest
    restart: always
    environment:
      EDGE: "1"
      EDGE_ID: "${EDGE_ID}"
      EDGE_KEY: "${EDGE_KEY}"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
      - /:/host
      - edge_agent_data:/data

volumes:
  edge_agent_data:
```

## Step 7: Verify Edge Agent Connection

```bash
# On Portainer server, check logs for agent connection
docker logs portainer 2>&1 | grep -i "edge\|agent\|connect"

# In Portainer UI:
# Environments → your edge environment → should show "Connected" status
# The green indicator means Edge Agent has checked in successfully

# Check Edge Agent logs on remote host
docker logs portainer_edge_agent --follow

# Expected: periodic heartbeat messages to edge.portainer.example.com
```

## Step 8: Secure Edge Agent Access with Cloudflare Access

Add authentication so only legitimate Edge Agents can reach the edge endpoint:

1. Create a **Service Token** in Cloudflare Access for Edge Agents
2. Create an Access Application for `edge.portainer.example.com`
3. Add a **Bypass** or **Service Auth** policy for the service token

```bash
# Edge Agents can use the service token in their connection
# Configure in the edge agent environment variables:
EDGE_ASYNC_TIMEOUT: "60s"
EDGE_CHECKIN_INTERVAL: "5s"
```

## Conclusion

Cloudflare Tunnel enables Portainer Edge Agents in remote or locked-down environments to communicate with the central Portainer server without any inbound firewall rules. The Edge Agent initiates outbound connections through Cloudflare's global network, making it ideal for environments behind strict firewalls, NAT, or in cloud regions where direct connectivity isn't available. Pair with Cloudflare Access service tokens to ensure only legitimate Edge Agents can reach your Portainer edge communication port.
