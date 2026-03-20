# How to Fix Endpoint Instability in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Endpoints, Stability

Description: Address Portainer endpoint instability where environments frequently switch between online and offline states, causing unreliable management and false alerts.

## Introduction

An unstable endpoint in Portainer flickers between online and offline states — it shows green for a few minutes, then goes red, then recovers on its own. This is different from a permanently failed connection. Instability is usually caused by network intermittency, resource exhaustion, agent health check failures, or snapshot timeout issues.

## Step 1: Identify the Pattern

```bash
# Check Portainer logs for repeated connection/disconnection messages
docker logs portainer 2>&1 | grep -i "endpoint\|connect\|disconnect\|timeout" | tail -50

# Look for patterns like:
# "Failed to query endpoint" followed by "Endpoint is back online"
# This indicates intermittent connectivity, not permanent failure
```

## Step 2: Check Agent Host Resources

An agent on an overloaded host will respond slowly or intermittently:

```bash
# SSH to the agent host and check resources
docker stats --no-stream  # Container CPU/RAM usage
free -h                    # Available RAM
df -h                      # Disk space
uptime                     # Load average
```

If the host is overloaded:

```bash
# Reduce container load or add resources
# Restart the most memory-hungry containers
docker stats --no-stream | sort -k 4 -hr | head -5

# Kill/restart problematic containers
docker restart <container-id>
```

## Step 3: Check Network Stability

```bash
# Run a ping test from Portainer server to agent (continuous)
ping -c 100 agent-host | tail -5

# Check for packet loss
# Expected: 0% packet loss
# If you see > 0%, there's network instability

# Test with larger packets (more realistic)
ping -s 1400 -c 50 agent-host

# Check for MTU issues
traceroute agent-host
```

## Step 4: Increase Snapshot Interval to Reduce Load

Frequent snapshots can overwhelm a slow or busy agent:

```bash
# Restart Portainer with a longer snapshot interval
docker stop portainer && docker rm portainer

docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval=120  # Every 2 minutes instead of 60s (default)
```

## Step 5: Fix Docker Daemon Instability on Agent Host

```bash
# Check Docker daemon health on the agent host
sudo systemctl status docker
journalctl -u docker --since "1 hour ago" | grep -i "error\|warn\|failed"

# Common Docker daemon issues causing agent instability:
# - Memory pressure (OOM killer hitting Docker)
# - Disk full
# - inotify watches exhausted

# Check for OOM events
dmesg | grep -i "oom\|killed" | tail -10

# Check inotify limits
cat /proc/sys/fs/inotify/max_user_watches
# If near the limit, increase:
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Step 6: Fix Agent Health Check Failures

```bash
# Check if the agent has health check failures
docker inspect portainer-agent | grep -A 10 '"Health"'

# If health checks are failing intermittently
# Adjust health check parameters when redeploying the agent:
docker stop portainer-agent && docker rm portainer-agent

docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  --health-cmd="nc -z localhost 9001 || exit 1" \
  --health-interval=30s \
  --health-retries=5 \
  --health-timeout=10s \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

## Step 7: Configure Portainer Server Timeout Settings

For slow or geographically distant agents:

```bash
# Portainer doesn't expose connection timeout directly, but
# you can adjust via the environment settings:
# In Portainer UI → Environments → Edit → Advanced settings
# Increase the snapshot interval for specific slow environments
```

## Step 8: Use a Stable DNS Name

If you're using a dynamic IP for the agent, the IP changing causes instability:

```bash
# Always use a hostname, not an IP address for environments
# Ensure the hostname resolves consistently
nslookup agent-hostname

# If using an IP that might change, use a DDNS service
# or a local DNS entry

# In /etc/hosts on the Portainer server host:
echo "192.168.1.50 agent-host" | sudo tee -a /etc/hosts
```

## Step 9: Check for Restart Loops on Agent

```bash
# Check if the agent is restarting frequently
docker inspect portainer-agent | grep '"RestartCount"'

# High restart count indicates a configuration problem
# View the restart history
docker events --filter container=portainer-agent --since 1h

# Fix restart loop by examining logs at startup
docker logs portainer-agent 2>&1 | head -30
```

## Step 10: Set Up Monitoring for Endpoint Status

Use the Portainer API to monitor endpoint status programmatically:

```bash
#!/bin/bash
# Check all endpoint statuses
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints | \
  jq '.[] | {name: .Name, status: .Status, url: .URL}'

# Status values: 1 = Online, 2 = Offline
```

## Conclusion

Endpoint instability in Portainer is almost always caused by resource exhaustion on the agent host, network packet loss, or the Docker daemon itself being unstable. Start with checking agent host resources (CPU, RAM, disk), then test network stability, and finally tune the snapshot interval to reduce the polling frequency if the agent is being overwhelmed.
