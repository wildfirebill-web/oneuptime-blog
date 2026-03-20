# How to Install Portainer Agent on Docker Standalone - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Agent, Docker, Remote Management, Installation

Description: Install and configure the Portainer Agent on a Docker standalone host to enable remote management from a central Portainer server.

## Introduction

The Portainer Agent is a lightweight service that runs on Docker hosts and enables remote management from a central Portainer server. Unlike direct socket or API connections, the agent handles the communication complexity and supports advanced features like browsing volumes and multi-host cluster management.

## Why Use an Agent

| Feature | Direct Socket | Docker API | Agent |
|---------|------------|-----------|-------|
| Local host only | Yes | No | No |
| Remote hosts | No | Yes | Yes |
| Volume browsing | No | No | Yes |
| No TLS setup needed | Yes | No | Yes |
| Swarm multi-node | No | No | Yes |

## Prerequisites

- Docker installed on the target host
- Port 9001 accessible from the Portainer server
- Portainer server already running

## Step 1: Install the Agent via Docker Run

```bash
docker run -d \
  --name portainer_agent \
  --restart always \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

## Step 2: Install via Docker Compose

```yaml
# docker-compose-agent.yml

version: "3.8"

services:
  agent:
    image: portainer/agent:latest
    container_name: portainer_agent
    restart: always
    ports:
      - "9001:9001"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    environment:
      # Optional: Set agent secret
      AGENT_SECRET: "shared-secret-with-portainer-server"
```

```bash
docker compose -f docker-compose-agent.yml up -d
```

## Step 3: Add the Agent-Managed Environment to Portainer

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints \
  -d '{
    "name": "Docker Host 1",
    "endpointCreationType": 2,
    "URL": "tcp://192.168.1.100:9001",
    "TLS": false
  }'
```

## Step 4: Verify Agent Connection

```bash
# Check agent is running
docker ps | grep portainer_agent

# Check agent logs
docker logs portainer_agent

# Test connectivity from Portainer host
nc -zv 192.168.1.100 9001
```

## Firewall Configuration

```bash
# Allow port 9001 from Portainer server only
sudo ufw allow from PORTAINER_SERVER_IP to any port 9001 proto tcp
```

## Agent Version Compatibility

The agent version should match the Portainer server version:

```bash
# Check current agent version
docker inspect portainer_agent | python3 -c "
import sys, json
c = json.load(sys.stdin)
print('Image:', c[0]['Config']['Image'])
"

# Update agent to match server
docker pull portainer/agent:latest
docker stop portainer_agent
docker rm portainer_agent
# Re-run docker run command above
```

## Conclusion

The Portainer Agent provides the most capable connection method for remote Docker hosts. It enables volume browsing, handles authentication, and supports Swarm cluster-wide management. Install it on each host you want to manage centrally, ensure port 9001 is open from the Portainer server, and add the environment in Portainer pointing to the agent's address.
