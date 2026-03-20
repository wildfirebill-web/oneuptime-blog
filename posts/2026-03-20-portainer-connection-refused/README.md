# How to Fix 'Connection Refused' Errors in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Networking, Self-Hosted

Description: Systematically diagnose and fix 'Connection Refused' errors in Portainer, whether they occur in the browser, between Portainer and agents, or between Portainer and Docker.

## Introduction

"Connection Refused" in Portainer can mean several different things depending on where it occurs: in the browser when accessing the Portainer UI, when Portainer tries to connect to a Docker endpoint, or when the Portainer Agent refuses connections from the server. This guide covers all three scenarios.

## Scenario 1: Browser Cannot Reach Portainer UI

### Symptoms

- Browser shows "ERR_CONNECTION_REFUSED"
- `curl http://your-host:9000` returns "Connection refused"

### Diagnostics

```bash
# 1. Check if Portainer is running

docker ps | grep portainer

# 2. Check the port binding
docker inspect portainer | grep -A 5 '"PortBindings"'

# 3. Check if anything is listening
ss -tlnp | grep 9000
```

### Fixes

```bash
# Portainer is not running - start it
docker start portainer

# Portainer crashed - check why and fix, then restart
docker logs portainer --tail 50
docker start portainer

# Port is not exposed - recreate with correct port mapping
docker stop portainer && docker rm portainer
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Scenario 2: Portainer Cannot Connect to Docker Socket

### Symptoms

- Portainer UI loads but shows "Connection refused" when listing containers
- Error: "Get /v1.44/containers/json: dial unix /var/run/docker.sock: connect: connection refused"

### Diagnostics

```bash
# Check Docker daemon status
systemctl status docker

# Check the socket exists
ls -la /var/run/docker.sock

# Test the socket directly
curl --unix-socket /var/run/docker.sock http://localhost/v1.44/version
```

### Fixes

```bash
# Docker is not running - start it
sudo systemctl start docker
sudo systemctl enable docker  # Ensure it starts on boot

# After Docker restarts, restart Portainer
docker restart portainer

# If the socket path is different (e.g., rootless Docker)
docker run -d \
  -p 9000:9000 \
  --name portainer \
  -v /run/user/1000/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Scenario 3: Portainer Cannot Connect to an Agent

### Symptoms

- Environment shows "Connection refused" or red status indicator
- Error: "dial tcp X.X.X.X:9001: connect: connection refused"

### Diagnostics

```bash
# From the Portainer server host, test connectivity to the agent
telnet agent-host 9001
# or
nc -zv agent-host 9001

# Check if the agent is running on the remote host
# (SSH to the agent host first)
docker ps | grep portainer-agent

# Check agent port binding
docker inspect portainer-agent | grep -A 5 '"PortBindings"'

# Check firewall on agent host
sudo ufw status
sudo firewall-cmd --list-ports  # RHEL/CentOS
```

### Fixes on the Agent Host

```bash
# Start the agent if not running
docker start portainer-agent

# If agent wasn't running - deploy it
docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# Open the firewall port
sudo ufw allow 9001/tcp
# or
sudo firewall-cmd --permanent --add-port=9001/tcp
sudo firewall-cmd --reload
```

## Scenario 4: Connection Refused on HTTPS Port 9443

```bash
# Check if Portainer is listening on 9443
ss -tlnp | grep 9443

# Test HTTPS connectivity (ignore cert errors for self-signed)
curl -k https://localhost:9443

# If HTTPS is not listening, Portainer may have been started with --http-disabled
# but without a valid TLS certificate
docker logs portainer | grep -i "tls\|cert\|ssl"
```

Fix for self-signed certificate issues:

```bash
# Stop and remove
docker stop portainer && docker rm portainer

# Run with auto-generated self-signed cert (default)
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Quick Diagnostic Checklist

```bash
#!/bin/bash
# Portainer connection diagnostic script

echo "=== Docker Status ==="
systemctl is-active docker

echo "=== Portainer Container Status ==="
docker inspect portainer --format='{{.State.Status}}' 2>/dev/null || echo "Not found"

echo "=== Port Bindings ==="
docker inspect portainer --format='{{json .HostConfig.PortBindings}}' 2>/dev/null

echo "=== Listening Ports ==="
ss -tlnp | grep -E "9000|9001|9443"

echo "=== Recent Logs ==="
docker logs portainer --tail 20 2>/dev/null
```

## Conclusion

"Connection refused" in Portainer always has a specific cause: the Portainer container isn't running, the port isn't exposed, Docker isn't running, or network/firewall rules are blocking the connection. Work through each scenario systematically - checking the container state, port bindings, Docker daemon health, and firewall rules - and you'll find the root cause quickly.
