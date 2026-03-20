# How to Fix Endpoint Instability in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Endpoint, Stability, Docker, Agent

Description: Learn how to diagnose and resolve endpoint instability in Portainer, where environments frequently toggle between online and offline states.

---

Endpoint instability - environments that randomly go offline and come back - is usually caused by snapshot timeouts, overloaded Docker hosts, or network congestion. This guide covers systematic diagnosis and remediation.

## Understanding Endpoint Health Checks

Portainer polls each endpoint every `--snapshot-interval` seconds (default 60s). If a snapshot fails or times out, the endpoint shows as offline. Repeated failures create the "flapping" behavior.

## Step 1: Check Snapshot Errors in Logs

```bash
# Look for snapshot-related errors

docker logs portainer 2>&1 | grep -i "snapshot\|endpoint\|error" | tail -50

# Common error patterns:
# "error creating snapshot" → agent unreachable during snapshot
# "context deadline exceeded" → snapshot timed out
# "connection reset by peer" → agent connection dropped
```

## Step 2: Check Agent Host Resource Usage

An overloaded Docker host causes slow responses that Portainer interprets as failures:

```bash
# Check CPU and memory on the agent host
top -b -n 1 | head -20

# Check Docker daemon load
systemctl status docker
journalctl -u docker --since "1 hour ago" | tail -30
```

## Step 3: Increase Snapshot Interval

Reduce polling frequency to give stressed hosts more breathing room:

```bash
# Restart Portainer with a longer snapshot interval (5 minutes)
docker run -d \
  --name portainer \
  --restart=always \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval 300
```

## Step 4: Check Network Quality

Intermittent packet loss between Portainer and the agent causes timeout failures:

```bash
# Run a prolonged ping to measure packet loss
ping -c 100 <agent-host-ip>

# Check for MTU issues (especially common on VPN or overlay networks)
ping -M do -s 1472 <agent-host-ip>
```

If packet loss is above 1%, investigate the network path. MTU mismatches cause TCP connections to hang silently.

## Step 5: Check Agent Health Loop

```bash
# Tail agent logs looking for repeated errors
docker logs -f portainer_agent 2>&1 | grep -i error

# If you see repeated "error processing snapshot" messages,
# the agent itself is struggling - check its resource usage
docker stats portainer_agent --no-stream
```

## Step 6: Upgrade Agent Version

Stability issues are sometimes caused by agent bugs that have been fixed in newer versions:

```bash
# Pull latest agent
docker pull portainer/agent:latest
docker stop portainer_agent && docker rm portainer_agent
# Redeploy agent with the same configuration
```
