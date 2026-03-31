# How to Update Portainer Agent to Match Server Version

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Agent, Update, Version Management, Maintenance

Description: Keep the Portainer Agent version synchronized with your Portainer server version to ensure compatibility and access to new features.

## Introduction

The Portainer Agent and Portainer server are versioned together. Running mismatched versions can cause connection failures or missing features. This guide covers updating agents on Docker standalone, Swarm, and Kubernetes.

## Checking Current Versions

```bash
# Check Portainer server version

TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/system/version \
  | python3 -m json.tool

# Check agent version
docker inspect portainer_agent | python3 -c "
import sys, json
c = json.load(sys.stdin)
print('Agent Image:', c[0]['Config']['Image'])
"
```

## Update Agent on Docker Standalone

```bash
# 1. Pull the new version
docker pull portainer/agent:latest
# Or specify version: portainer/agent:2.21.0

# 2. Stop and remove old agent
docker stop portainer_agent
docker rm portainer_agent

# 3. Run new agent with same configuration
docker run -d \
  --name portainer_agent \
  --restart always \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# 4. Verify new version
docker inspect portainer_agent | grep -i "Image"
```

## Update Agent on Docker Swarm

```bash
# Update the global agent service to new image
docker service update \
  --image portainer/agent:latest \
  --update-parallelism 1 \
  --update-delay 10s \
  portainer-agent_agent

# Monitor the update
docker service ps portainer-agent_agent

# Check all replicas updated
docker service ls | grep agent
```

## Update Agent on Kubernetes via Helm

```bash
# Update Helm repos
helm repo update

# Check available versions
helm search repo portainer/portainer-agent --versions | head -10

# Upgrade to latest
helm upgrade portainer-agent \
  -n portainer \
  portainer/portainer-agent

# Or upgrade to specific version
helm upgrade portainer-agent \
  -n portainer \
  portainer/portainer-agent \
  --version 1.0.58

# Monitor rollout
kubectl rollout status daemonset/portainer-agent -n portainer
```

## Automating Agent Updates

```bash
#!/bin/bash
# update-portainer-agent.sh

AGENTS=("192.168.1.50" "192.168.1.51" "192.168.1.52")
SSH_USER="ubuntu"

for agent in "${AGENTS[@]}"; do
  echo "Updating agent on $agent..."
  ssh "$SSH_USER@$agent" '
    docker pull portainer/agent:latest
    docker stop portainer_agent
    docker rm portainer_agent
    docker run -d \
      --name portainer_agent \
      --restart always \
      -p 9001:9001 \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v /var/lib/docker/volumes:/var/lib/docker/volumes \
      portainer/agent:latest
    docker inspect portainer_agent --format "{{.Config.Image}}"
  '
  echo "Agent updated on $agent"
done
```

## Conclusion

Keeping agent versions synchronized with the Portainer server is an important maintenance task. Pin agent versions to match the server version in production (e.g., `portainer/agent:2.21.0` not `latest`), and update both server and agents together during maintenance windows to avoid version skew.
