# How to Set Container Resource Limits (CPU and Memory) in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Performance, DevOps

Description: Learn how to set CPU and memory resource limits and reservations on Docker containers in Portainer to prevent resource contention.

## Introduction

Without resource limits, a single runaway container can consume all available CPU or memory on a host, starving other containers and degrading system stability. Portainer provides a straightforward interface for setting both hard limits and soft reservations on container resources.

## Prerequisites

- Portainer installed with a connected Docker environment
- Basic understanding of CPU and memory concepts

## Understanding Limits vs. Reservations

Docker supports two types of resource constraints:

| Type | CPU | Memory | Description |
|------|-----|--------|-------------|
| **Limit** (hard) | `--cpus` | `--memory` | Maximum allowed — enforced strictly |
| **Reservation** (soft) | `--cpu-shares` | `--memory-reservation` | Guaranteed minimum — used for scheduling |

- **Limit**: Container cannot exceed this amount. If memory limit is hit, the container is OOM-killed.
- **Reservation**: Guarantees this amount is available when the system is under pressure.

## Step 1: Set Resources During Container Creation

1. Navigate to **Containers > Add container**.
2. Set the container name and image.
3. Scroll to the **Resources** tab (sometimes labeled **Runtime & Resources**).

## Step 2: Configure Memory Limits

### Memory Limit (Hard Limit)

Enter the maximum memory the container can use:

```
Memory limit: 512   (MB)
# Container will be OOM-killed if it exceeds 512 MB
```

Common values:
- Small service: 128-256 MB
- Medium application: 512 MB - 1 GB
- Large application (Java, Node): 1-2 GB
- Database: 2-4 GB

### Memory Reservation (Soft Limit)

Enter the memory reserved/guaranteed for the container:

```
Memory reservation: 256   (MB)
# Docker scheduler ensures 256 MB is available for this container
```

The reservation should be less than or equal to the memory limit.

### Memory + Swap

```
Memory limit:  512 MB
Memory swap:   1024 MB   (total memory + swap)
# Swap available = 1024 - 512 = 512 MB of swap
```

Setting swap to the same value as the memory limit disables swap:

```
Memory limit: 512 MB
Memory swap:  512 MB   # No additional swap available
```

## Step 3: Configure CPU Limits

### CPU Limit

Specify how many CPUs the container can use:

```
CPU limit: 0.5
# Container can use up to 50% of one CPU core

CPU limit: 2.0
# Container can use up to 2 full CPU cores
```

This maps to Docker's `--cpus` flag.

### CPU Reservation

The minimum CPU guaranteed to the container:

```
CPU reservation: 0.25
# Container always gets at least 25% of one CPU
```

## Docker Compose Equivalent

In Portainer Stacks (docker-compose.yml):

```yaml
version: "3.8"

services:
  # Web application with resource limits
  web:
    image: myorg/webapp:latest
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '0.5'       # Max 0.5 CPU cores
          memory: 512M      # Max 512 MB RAM
        reservations:
          cpus: '0.25'      # Reserved 0.25 CPU cores
          memory: 256M      # Reserved 256 MB RAM

  # Database with higher limits
  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M

  # Background worker with CPU limit
  worker:
    image: myorg/worker:latest
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 256M
```

Note: For standalone containers (not Swarm), use `--cpus` and `--memory` flags directly. In compose for standalone, place limits under the service:

```yaml
services:
  app:
    image: myapp:latest
    mem_limit: 512m
    memswap_limit: 512m
    cpus: 0.5
```

## Step 4: Monitor Resource Usage

After setting limits, verify containers are within bounds:

1. In Portainer, navigate to **Containers**.
2. Click **Stats** next to a container.
3. View real-time CPU %, memory usage vs. limit, and network I/O.

```bash
# Equivalent CLI command:
docker stats --no-stream --format \
  "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}" \
  my-container
```

## Handling OOM (Out of Memory) Events

When a container exceeds its memory limit, Docker kills it with an OOM event:

```bash
# Check if a container was OOM-killed:
docker inspect my-container | jq '.[].State.OOMKilled'
# Returns: true if OOM-killed

# In Portainer: check container details for OOM status
# Navigate to container > Inspect tab > State.OOMKilled
```

If a container is being OOM-killed:
1. Increase the memory limit.
2. Or optimize the application's memory usage.
3. Check container logs for memory leak patterns.

## Best Practices

- **Always set memory limits** on production containers to prevent runaway processes.
- **Set realistic reservations** based on baseline memory usage observed in testing.
- **Use CPU limits for batch workers** to prevent them from starving real-time services.
- **Monitor for OOM kills** — they indicate the container needs more memory or has a leak.
- **Leave headroom on the host** — don't reserve 100% of host resources across containers.

## Conclusion

Resource limits in Portainer protect your host from container resource abuse and ensure fair resource distribution across workloads. By setting both hard limits and soft reservations, you create a stable multi-tenant container environment where each service gets what it needs without disrupting others.
