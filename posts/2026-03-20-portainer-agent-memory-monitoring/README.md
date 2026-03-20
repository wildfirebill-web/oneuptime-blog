# How to Monitor Agent Memory Usage in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Agent, Memory, Monitoring, Performance

Description: Monitor and manage Portainer Agent memory consumption to prevent resource issues on Docker hosts.

## Introduction

The Portainer Agent is lightweight but can consume increasing memory in high-load environments or when managing large numbers of containers. Monitoring agent memory usage prevents out-of-memory situations that would disrupt container management.

## Checking Agent Memory Usage

```bash
# Real-time memory stats
docker stats portainer_agent --no-stream
# Shows: CPU %, MEM USAGE/LIMIT, MEM %

# Historical stats (watch mode)
docker stats portainer_agent

# Detailed memory breakdown
docker exec portainer_agent cat /proc/meminfo | head -20
```

## Setting Memory Limits on the Agent

Prevent the agent from consuming excessive memory by setting limits:

```bash
# Set memory limit to 256MB
docker run -d \
  --name portainer_agent \
  --restart always \
  -p 9001:9001 \
  --memory="256m" \
  --memory-swap="512m" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

```yaml
# docker-compose.yml
services:
  agent:
    image: portainer/agent:latest
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.25'
        reservations:
          memory: 64M
```

## Monitoring with Docker Stats API

```bash
# Get memory stats via Docker API
curl -s --unix-socket /var/run/docker.sock \
  "http://localhost/containers/portainer_agent/stats?stream=false" \
  | python3 -c "
import sys, json
stats = json.load(sys.stdin)
mem = stats['memory_stats']
used = mem['usage'] / 1024 / 1024
limit = mem['limit'] / 1024 / 1024
print(f'Memory: {used:.1f}MB / {limit:.1f}MB ({used/limit*100:.1f}%)')
"
```

## Alerting on High Memory Usage

```bash
#!/bin/bash
# check-agent-memory.sh

THRESHOLD_MB=200
CONTAINER_NAME="portainer_agent"

MEM_USAGE=$(docker stats $CONTAINER_NAME --no-stream --format "{{.MemUsage}}" \
  | awk '{print $1}' \
  | sed 's/MiB//')

if (( $(echo "$MEM_USAGE > $THRESHOLD_MB" | bc -l) )); then
  echo "WARNING: Agent memory usage ${MEM_USAGE}MB exceeds threshold ${THRESHOLD_MB}MB"
  # Add alerting logic here (email, Slack, PagerDuty)
fi
```

## Reducing Agent Memory Footprint

If the agent uses excessive memory:

```bash
# 1. Restart the agent periodically via cron
0 3 * * 0 docker restart portainer_agent  # Weekly restart Sunday 3am

# 2. Reduce log verbosity
docker run -e LOG_LEVEL=ERROR portainer/agent:latest

# 3. Check for memory leaks - track over time
for i in $(seq 1 10); do
  docker stats portainer_agent --no-stream --format "{{.MemUsage}}"
  sleep 60
done
```

## Conclusion

Portainer Agent memory usage is typically modest (50-150MB) but can grow with many containers or high snapshot frequency. Set memory limits to protect host stability, monitor usage trends, and restart the agent on a schedule if memory growth is observed. The Docker stats API provides the most detailed memory breakdown for diagnosing consumption patterns.
