# How to Configure Async Edge Agent Ping and Snapshot Frequency

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge Agent, Async, Snapshot, Ping Interval, IoT

Description: Fine-tune the async Edge Agent's ping interval and snapshot frequency in Portainer for optimal performance in bandwidth-constrained environments.

## Introduction

The async Edge Agent has three separate timing controls: the ping interval (heartbeat), the command check interval, and the snapshot interval. Understanding each and configuring them appropriately is key to building efficient edge deployments.

## The Three Async Intervals Explained

### EDGE_PING_INTERVAL (Heartbeat)

How often the agent sends a heartbeat to Portainer. This keeps the agent registered as "online":

```bash
-e EDGE_PING_INTERVAL=30  # Send heartbeat every 30 seconds
```

**Effect on UI**: The "last seen" timestamp in Portainer updates this frequently.

### EDGE_CMD_INTERVAL (Command Polling)

How often the agent checks for queued commands from Portainer:

```bash
-e EDGE_CMD_INTERVAL=10  # Check for commands every 10 seconds
```

**Effect on UI**: Time between issuing a command and it executing on the remote device.

### EDGE_SNAPSHOT_INTERVAL (State Reporting)

How often the agent sends a full environment snapshot (container list, status, resource usage):

```bash
-e EDGE_SNAPSHOT_INTERVAL=300  # Send snapshot every 5 minutes
```

**Effect on UI**: How fresh the container status information is in Portainer.

## Configuring All Intervals

```yaml
# docker-compose.yml
version: "3.8"

services:
  edge-agent:
    image: portainer/agent:latest
    environment:
      EDGE: "1"
      EDGE_ID: "remote-device-001"
      EDGE_KEY: "your-edge-key-here"
      EDGE_ASYNC: "1"
      
      # Heartbeat: frequent, small payload
      EDGE_PING_INTERVAL: "30"
      
      # Commands: more frequent for better responsiveness
      EDGE_CMD_INTERVAL: "15"
      
      # Snapshots: less frequent, larger payload
      EDGE_SNAPSHOT_INTERVAL: "300"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
      - /var/run/portainer:/var/run/portainer
    restart: always
```

## Monitoring Interval Effectiveness

```bash
# Watch agent logs to see actual check-in times
docker logs portainer_async_agent -f | grep -E "ping|snapshot|command|check"

# In Portainer UI: Monitor the "Last Check-in" timestamp
# It should update at approximately EDGE_PING_INTERVAL frequency
```

## Tuning for Different Connectivity Types

### High-Speed Fiber (Office Branch)
```bash
EDGE_PING_INTERVAL=10
EDGE_CMD_INTERVAL=5
EDGE_SNAPSHOT_INTERVAL=60
```

### 4G/LTE Connection
```bash
EDGE_PING_INTERVAL=60
EDGE_CMD_INTERVAL=30
EDGE_SNAPSHOT_INTERVAL=300
```

### Satellite Internet (High Latency)
```bash
EDGE_PING_INTERVAL=300
EDGE_CMD_INTERVAL=60
EDGE_SNAPSHOT_INTERVAL=1800
```

### Infrequent Check-In (Solar/Battery Powered)
```bash
EDGE_PING_INTERVAL=3600
EDGE_CMD_INTERVAL=300
EDGE_SNAPSHOT_INTERVAL=7200
```

## Impact of Long Intervals on User Experience

With a 1-hour ping interval:
- Portainer shows the device as "offline" between check-ins
- Commands may wait up to 1 hour to execute
- Snapshots may be hours old

Portainer Business Edition shows a "last seen" timestamp and marks devices offline after 2x the ping interval without contact.

## Conclusion

The three async intervals give fine-grained control over the trade-off between responsiveness and bandwidth. Set the snapshot interval highest (it has the largest payload), the ping interval in the middle, and the command interval lowest for the best balance of efficiency and responsiveness.
