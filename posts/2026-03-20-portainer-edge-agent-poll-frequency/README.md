# How to Configure Edge Agent Poll Frequency - Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge Agent, Poll Interval, Configuration, Performance

Description: Tune the Portainer Edge Agent's polling interval to balance command responsiveness against network bandwidth usage.

## Introduction

The Edge Agent's poll frequency determines how often it checks in with the Portainer server for new commands. A lower interval means faster command execution but higher bandwidth consumption. This guide covers configuring poll frequency for different use cases.

## Standard Mode Poll Interval

In standard mode, the `EDGE_POLL_INTERVAL` controls how frequently the agent checks for new commands:

```bash
# Default: 5 seconds

docker run -d \
  -e EDGE=1 \
  -e EDGE_ID=device-id \
  -e EDGE_KEY=edge-key \
  -e EDGE_POLL_INTERVAL=5 \
  portainer/agent:latest
```

## Async Mode Intervals

Async mode has three independent interval settings:

```bash
docker run -d \
  -e EDGE=1 \
  -e EDGE_ID=device-id \
  -e EDGE_KEY=edge-key \
  -e EDGE_ASYNC=1 \
  -e EDGE_PING_INTERVAL=30 \        # Heartbeat interval (seconds)
  -e EDGE_CMD_INTERVAL=10 \         # Command check interval (seconds)
  -e EDGE_SNAPSHOT_INTERVAL=300 \   # State snapshot interval (seconds)
  portainer/agent:latest
```

## Interval Selection Guide

| Scenario | Ping | Commands | Snapshot |
|----------|------|---------|---------|
| Low-latency, good connectivity | 5s | 5s | 60s |
| Standard office/branch | 30s | 10s | 300s |
| Metered/cellular connection | 60s | 30s | 600s |
| Very remote / satellite | 300s | 60s | 3600s |
| IoT with daily check-in | 3600s | 300s | 86400s |

## Updating Poll Interval on Running Agent

```bash
# To change poll interval, stop and recreate the container
docker stop portainer_edge_agent
docker rm portainer_edge_agent

# Restart with new interval
docker run -d \
  --name portainer_edge_agent \
  -e EDGE=1 \
  -e EDGE_ID=device-id \
  -e EDGE_KEY=edge-key \
  -e EDGE_ASYNC=1 \
  -e EDGE_PING_INTERVAL=60 \
  -e EDGE_CMD_INTERVAL=20 \
  -e EDGE_SNAPSHOT_INTERVAL=600 \
  portainer/agent:latest
```

## Bandwidth Estimation

Calculate monthly bandwidth usage:

```bash
# Approximate payload sizes:
# Ping: ~1KB per check-in
# Command poll: ~2KB per check
# Snapshot: ~10-100KB per snapshot (depends on container count)

python3 << 'EOF'
PING_INTERVAL = 30          # seconds
CMD_INTERVAL = 10           # seconds
SNAPSHOT_INTERVAL = 300     # seconds
DAYS = 30

PING_KB = 1
CMD_KB = 2
SNAPSHOT_KB = 50  # average

pings_per_day = 86400 / PING_INTERVAL
cmds_per_day = 86400 / CMD_INTERVAL
snapshots_per_day = 86400 / SNAPSHOT_INTERVAL

daily_mb = (pings_per_day * PING_KB + cmds_per_day * CMD_KB + snapshots_per_day * SNAPSHOT_KB) / 1024
monthly_mb = daily_mb * DAYS

print(f"Pings/day: {pings_per_day:.0f} ({pings_per_day * PING_KB / 1024:.1f} MB/day)")
print(f"Commands/day: {cmds_per_day:.0f} ({cmds_per_day * CMD_KB / 1024:.1f} MB/day)")
print(f"Snapshots/day: {snapshots_per_day:.0f} ({snapshots_per_day * SNAPSHOT_KB / 1024:.1f} MB/day)")
print(f"Total: {daily_mb:.1f} MB/day, {monthly_mb:.0f} MB/month")
EOF
```

## Conclusion

Poll frequency configuration is a key tuning parameter for edge deployments. Start with the defaults (5s for standard mode) and increase intervals if bandwidth or cost is a concern. For IoT or remote monitoring scenarios where hours-long response times are acceptable, use very long intervals to minimize data usage while maintaining management capability.
