# How to Monitor Agent Memory Usage in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Agent, Monitoring, Memory, Performance

Description: Track and manage Portainer Agent memory consumption to ensure agents don't impact host system performance.

---

How to Monitor Agent Memory Usage in Portainer is an important operational task for maintaining reliable Portainer infrastructure.

## Overview

The Portainer Agent communicates with the Portainer server on TCP port 9001. Proper configuration and troubleshooting of the agent is essential for uninterrupted container management.

## Common Configuration Steps

```bash
# Check agent container status

docker ps --filter name=portainer_agent

# View agent logs for errors
docker logs portainer_agent --tail 50 2>&1

# Verify port 9001 is listening
ss -tlnp | grep 9001
# or
netstat -tlnp | grep 9001
```

## Network Connectivity Test

```bash
# From the Portainer server, test connectivity to the agent
nc -zv <agent-host-ip> 9001
# Expected: Connection succeeded

# Test with curl (should get a response)
curl -k https://<agent-host-ip>:9001 2>&1 | head -5
```

## Firewall Configuration

```bash
# Allow port 9001 (UFW)
sudo ufw allow from <portainer-server-ip> to any port 9001 proto tcp

# Allow port 9001 (firewalld)
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<portainer-server-ip>" port port="9001" protocol="tcp" accept'
sudo firewall-cmd --reload

# IPTables rule
sudo iptables -A INPUT -s <portainer-server-ip> -p tcp --dport 9001 -j ACCEPT
```

## SELinux Context Fix (RHEL/CentOS)

```bash
# Check for SELinux denials
sudo ausearch -c 'docker' --raw | audit2allow -M portainer-agent
sudo semodule -i portainer-agent.pp

# Or temporarily disable enforcement for testing
sudo setenforce 0

# Add correct context for Docker socket
sudo chcon -Rt svirt_sandbox_file_t /var/run/docker.sock
```

## Agent Version Compatibility

```bash
# Check current agent version
docker inspect portainer_agent --format '{{.Config.Image}}'

# Check Portainer server version
curl -s https://localhost:9443/api/status --insecure | python3 -c "
import sys, json
status = json.load(sys.stdin)
print(f'Server version: {status.get(\"Version\", \"unknown\")}')
"

# Update agent to match server version
docker stop portainer_agent && docker container rm portainer_agent
docker pull portainer/agent:latest
docker run -d -p 9001:9001 --name portainer_agent --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

---

*Keep agent-based environments healthy with proactive monitoring from [OneUptime](https://oneuptime.com).*
