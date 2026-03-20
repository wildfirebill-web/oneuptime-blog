# How to Install Portainer Agent on Unraid

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Unraid, Docker, Portainer Agent, Multi-Host, Self-Hosted

Description: Install the Portainer Agent on your Unraid server to manage it from a central Portainer Business Edition or CE instance on another host.

## Introduction

The Portainer Agent allows you to manage a remote Docker host from a central Portainer instance. This is ideal for home labs where you want one Portainer dashboard managing multiple machines — including your Unraid server — without exposing the Docker socket directly.

## Prerequisites

- Unraid 6.11 or later with Docker enabled
- A central Portainer CE or BE instance running on another host
- Network connectivity between the Portainer server and the Unraid host
- Port 9001 available on the Unraid host

## Step 1: Install the Portainer Agent on Unraid

### Via Unraid Docker UI

1. Navigate to the **Docker** tab in Unraid WebUI
2. Click **Add Container**
3. Fill in:
   - **Name**: portainer-agent
   - **Repository**: portainer/agent:latest
   - **Network Type**: Bridge
   - **Restart Policy**: Unless Stopped

4. Add Port mapping:
   - Container Port: `9001`
   - Host Port: `9001`
   - Protocol: TCP

5. Add Volume for Docker socket:
   - Container Path: `/var/run/docker.sock`
   - Host Path: `/var/run/docker.sock`
   - Access Mode: Read/Write

6. Add Volume for Docker volumes:
   - Container Path: `/var/lib/docker/volumes`
   - Host Path: `/var/lib/docker/volumes`
   - Access Mode: Read/Write

7. Click **Apply**

### Via SSH

```bash
# SSH into Unraid
ssh root@<unraid-ip>

# Deploy the Portainer Agent
docker run -d \
  --name portainer-agent \
  --restart=unless-stopped \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# Verify it's running
docker ps | grep portainer-agent
```

## Step 2: Configure Unraid Firewall

Allow port 9001 inbound from your Portainer server:

If using Unraid's network settings or a separate firewall, allow TCP port `9001` from the IP address of your central Portainer server.

## Step 3: Add the Unraid Host to Central Portainer

On your **central Portainer instance**:

1. Navigate to **Environments > Add environment**
2. Select **Docker Standalone**
3. Choose **Agent**
4. Fill in:
   - **Name**: Unraid-Server (or your preferred name)
   - **Environment URL**: `<unraid-ip>:9001`
5. Click **Connect**

Portainer will verify the connection and add the Unraid host as a managed environment.

## Step 4: Verify the Connection

In Portainer, navigate to **Environments** and confirm the Unraid entry shows as **Up**. Click on it to see all containers, volumes, networks, and images on the Unraid host.

## Step 5: Managing Unraid from Portainer

Once connected, you can:

- **View and manage containers** started by Unraid's Docker manager
- **Deploy stacks** that Unraid doesn't natively support
- **Monitor resource usage** (CPU, memory, network) per container
- **Access container logs and consoles** directly from Portainer

### Important Note on Coexistence

Portainer Agent does not replace Unraid's Docker manager. Both can manage containers independently. To avoid conflicts:

- Use Unraid's Docker tab for apps installed via Community Applications
- Use Portainer stacks for multi-container applications
- Avoid stopping/removing Unraid-managed containers from Portainer

## Agent Configuration Options

You can customize the agent behavior with environment variables:

```bash
docker run -d \
  --name portainer-agent \
  --restart=unless-stopped \
  -p 9001:9001 \
  -e AGENT_CLUSTER_ADDR=<unraid-ip>    # For Swarm mode
  -e LOG_LEVEL=DEBUG                    # Enable debug logging
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

## Updating the Portainer Agent

```bash
# SSH into Unraid
ssh root@<unraid-ip>

# Stop and remove old agent
docker stop portainer-agent && docker rm portainer-agent

# Pull latest
docker pull portainer/agent:latest

# Redeploy
docker run -d \
  --name portainer-agent \
  --restart=unless-stopped \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

## Conclusion

The Portainer Agent on Unraid enables centralized management of your Unraid server alongside other Docker hosts in a single Portainer dashboard. This is particularly valuable in home labs with multiple machines, where switching between management interfaces becomes tedious. The agent uses a secure channel that avoids direct Docker socket exposure over the network.
