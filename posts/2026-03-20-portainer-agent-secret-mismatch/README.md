# How to Fix Agent Secret Mismatch Between Server and Agent

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Agent, Security, Authentication

Description: Resolve authentication failures between Portainer server and agent caused by AGENT_SECRET mismatches, including how to correctly configure and synchronize secrets.

## Introduction

When using `AGENT_SECRET` for secure agent-to-server communication, both sides must use the exact same secret. A mismatch results in the agent rejecting connections from the Portainer server (or vice versa), manifesting as a "Unable to connect to agent" or authentication error. This guide explains how to diagnose and fix this.

## How AGENT_SECRET Works

The `AGENT_SECRET` is a pre-shared key used to authenticate connections between the Portainer server and the Agent. When set:
- The Agent only accepts connections from a Portainer server that sends the matching secret
- The Portainer server sends the secret when connecting to the environment
- Both must be identical — case-sensitive

## Step 1: Check the Agent's Secret

```bash
# Check what secret the agent is configured with
docker inspect portainer-agent | grep -i "AGENT_SECRET"

# Example output showing the secret is set:
# "AGENT_SECRET=mysecret123"

# Check via environment
docker exec portainer-agent env | grep AGENT_SECRET
```

## Step 2: Check the Portainer Server's Environment Configuration

1. In Portainer UI, go to **Environments**
2. Click the affected environment to **Edit**
3. Scroll to the **Agent** section
4. Look for the **Agent secret** field

If the field is empty but the agent has a secret set, that's the mismatch.

## Step 3: Fix — Update the Environment in Portainer

1. In **Environments** → Edit the failing environment
2. Find the **Agent secret** field
3. Enter the exact same value as `AGENT_SECRET` on the agent
4. Click **Save**
5. Portainer will retry the connection immediately

## Step 4: Fix — Update the Agent Secret

If you want to change the secret to match what Portainer expects:

```bash
# Stop and remove the current agent
docker stop portainer-agent
docker rm portainer-agent

# Redeploy with the correct secret
docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  -e AGENT_SECRET="the-secret-portainer-expects" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# Verify the secret is set
docker exec portainer-agent env | grep AGENT_SECRET
```

## Step 5: Set Up a Strong Secret from Scratch

If you're configuring everything fresh:

```bash
# Generate a strong random secret
openssl rand -hex 32
# Example: 4a8f2c1e9d3b7a6e5f8c2d1b4e7a9f3c2b1d8e7f6a5c4b3d2e1f9a8b7c6d5e4f

# Deploy agent with this secret
docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  -e AGENT_SECRET="4a8f2c1e9d3b7a6e5f8c2d1b4e7a9f3c2b1d8e7f6a5c4b3d2e1f9a8b7c6d5e4f" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# In Portainer, when adding the environment:
# URL: agent-host:9001
# Agent secret: 4a8f2c1e9d3b7a6e5f8c2d1b4e7a9f3c2b1d8e7f6a5c4b3d2e1f9a8b7c6d5e4f
```

## Step 6: Fix for Docker Compose Deployments

```yaml
services:
  portainer-agent:
    image: portainer/agent:latest
    ports:
      - "9001:9001"
    environment:
      # Must match what's configured in Portainer Environments settings
      AGENT_SECRET: "your-shared-secret-here"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    restart: unless-stopped
```

## Step 7: Fix for Docker Swarm Global Service

```bash
# Update the service environment variable
docker service update \
  --env-add AGENT_SECRET="the-correct-secret" \
  portainer_portainer-agent

# Verify the update
docker service inspect portainer_portainer-agent | grep AGENT_SECRET
```

## Step 8: Debug Authentication Failures

```bash
# Enable debug logging on the agent to see authentication details
docker stop portainer-agent && docker rm portainer-agent

docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  -e AGENT_SECRET="your-secret" \
  -e LOG_LEVEL=DEBUG \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# Watch for authentication messages
docker logs -f portainer-agent | grep -i "auth\|secret\|signature\|invalid"
```

## Step 9: Common Mistakes to Avoid

```bash
# WRONG: Secret with trailing newline (common when using echo)
echo "mysecret" > /tmp/secret  # Adds newline — don't do this
AGENT_SECRET=$(cat /tmp/secret)  # Will include the newline

# CORRECT: Use printf or specify exact value
printf "mysecret" > /tmp/secret
AGENT_SECRET=$(cat /tmp/secret)

# Or just specify directly in the run command:
-e AGENT_SECRET="mysecret"

# WRONG: Different case
# Agent: AGENT_SECRET="MySecret"
# Portainer config: Agent secret: "mysecret"  ← case mismatch

# CORRECT: Exact same string
# Both use: "MySecret"
```

## Step 10: Remove the Secret (Disable Authentication)

For internal networks where you don't need secret-based auth:

```bash
# Deploy agent WITHOUT a secret (allows any Portainer server to connect)
docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# In Portainer, leave the "Agent secret" field empty
```

> **Security Note**: Only use this approach on isolated, trusted networks. Without a secret, any Portainer instance can connect to your agent.

## Conclusion

Agent secret mismatches are simple to fix but easy to miss: both the `AGENT_SECRET` environment variable on the agent and the "Agent secret" field in Portainer's environment configuration must contain the exact same string. Always generate a strong random secret with `openssl rand -hex 32`, set it in both places simultaneously, and avoid trailing newlines or whitespace that can silently corrupt the value.
