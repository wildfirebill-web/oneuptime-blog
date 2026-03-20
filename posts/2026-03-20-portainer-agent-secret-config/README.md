# How to Configure the Agent Secret Between Portainer Server and Agent

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Agent, Security, Authentication, Secret

Description: Configure a shared agent secret to secure the communication channel between the Portainer server and its agents.

## Introduction

By default, the Portainer Agent accepts connections from any Portainer server. To prevent unauthorized servers from connecting to your agents, you can configure a shared secret. Both the Portainer server and agents must use the same secret for communication to succeed.

## Setting the Agent Secret on the Agent Side

```bash
# Docker run
docker run -d \
  --name portainer_agent \
  --restart always \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -e AGENT_SECRET=your-shared-secret-here \
  portainer/agent:latest
```

```yaml
# docker-compose.yml
services:
  agent:
    image: portainer/agent:latest
    environment:
      AGENT_SECRET: "your-shared-secret-here"
    ports:
      - "9001:9001"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
```

## Setting the Agent Secret on the Portainer Server Side

When adding an environment that uses an agent with a secret, provide the secret in the connection settings:

### Via Web UI

1. Environments → Add environment → Docker Standalone or Swarm
2. Select **Agent**
3. Enter the agent URL
4. Enter the **Agent secret** in the provided field

### Via API

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
    "name": "Secured Agent Host",
    "endpointCreationType": 2,
    "URL": "tcp://agent-host.example.com:9001",
    "TLS": false,
    "AgentSecret": "your-shared-secret-here"
  }'
```

## Generating a Strong Secret

```bash
# Generate a cryptographically random secret
openssl rand -hex 32
# Example: a3f9c2d1e4b5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1

# Or use Python
python3 -c "import secrets; print(secrets.token_hex(32))"
```

## Rotating the Agent Secret

```bash
# 1. Stop the agent
docker stop portainer_agent

# 2. Update the agent with new secret
docker rm portainer_agent
docker run -d \
  --name portainer_agent \
  -e AGENT_SECRET=new-stronger-secret \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  portainer/agent:latest

# 3. Update the environment in Portainer with new secret
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1 \
  -d '{"AgentSecret": "new-stronger-secret"}'
```

## Verifying Secret Authentication

```bash
# Agent log should show authenticated connections
docker logs portainer_agent | grep -i "auth\|secret\|connect"
```

## Conclusion

The agent secret provides an additional authentication layer between Portainer and its agents. Use a unique, cryptographically random secret per environment, store it securely, and rotate it periodically. All agents in a cluster (Swarm) should use the same secret.
