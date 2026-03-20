# How to Scale Services in Portainer on Docker Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Scaling, Services, DevOps

Description: Learn how to scale Docker Swarm services up and down using Portainer's UI for handling variable workloads.

## Introduction

One of Docker Swarm's core strengths is the ability to scale services horizontally — adding or removing replicas to match demand. Portainer makes this process visual and immediate. This guide covers scaling services from the Portainer UI, understanding the scaling process, and best practices for production scaling.

## Prerequisites

- Portainer installed on Docker Swarm
- At least one service running on the cluster
- Admin or operator access

## Method 1: Scale from the Services List

The fastest way to scale is directly from the services list:

1. In Portainer, navigate to **Services**
2. Find the service you want to scale
3. In the **Scheduling** column, you will see the current replica count
4. Click the **Scale** icon (up/down arrows) next to the replica count
5. Enter the new number of replicas
6. Click **Apply**

Portainer immediately sends the scale command to the Swarm manager.

## Method 2: Scale from the Service Detail View

For more context when scaling:

1. Click on the service name to open details
2. In the service detail view, find the **Replicas** field
3. Click **Edit this service**
4. Update the replica count under **Scaling**
5. Click **Update the service**

## Method 3: Scale Multiple Services at Once

From the services list:

1. Check the checkboxes next to multiple services
2. Click **Scale** in the actions toolbar
3. Enter the desired replica count for each selected service
4. Confirm

## Understanding the Scale Process

When you increase replicas from 2 to 5:

```
Before scaling:
  web.1 → worker-01 (Running)
  web.2 → worker-02 (Running)

Scaling command issued: 5 replicas

After scaling:
  web.1 → worker-01 (Running)
  web.2 → worker-02 (Running)
  web.3 → worker-03 (Running)  ← New
  web.4 → worker-01 (Running)  ← New
  web.5 → worker-02 (Running)  ← New
```

The Swarm scheduler distributes new tasks across nodes based on current load and placement constraints.

## Verifying the Scale Operation

```bash
# Monitor scale progress from CLI
watch docker service ps web-frontend

# Confirm final state
docker service ls --filter name=web-frontend
```

In Portainer, refresh the Services page to see updated replica counts and task status.

## Scale to Zero

Scaling to 0 replicas stops all tasks without removing the service:

```bash
# Scale to zero from CLI
docker service scale web-frontend=0

# Scale back up
docker service scale web-frontend=3
```

This is useful for:
- Temporarily suspending a service for maintenance
- Freeing resources without deleting service configuration

From Portainer, set replicas to `0` using the scale field.

## Auto-Scaling Strategies

Docker Swarm does not have built-in auto-scaling, but you can implement it with external tools:

### Option 1: Shell Script Auto-Scaler

```bash
#!/bin/bash
# simple-autoscaler.sh
SERVICE="web-frontend"
MIN_REPLICAS=2
MAX_REPLICAS=10
SCALE_UP_THRESHOLD=80    # CPU percentage
SCALE_DOWN_THRESHOLD=30  # CPU percentage

while true; do
    # Get current CPU usage (simplified)
    CPU=$(docker stats --no-stream --format "{{.CPUPerc}}" \
        $(docker service ps $SERVICE -q --filter desired-state=running) | \
        awk -F% '{sum += $1; count++} END {print sum/count}')

    CURRENT=$(docker service ls --filter name=$SERVICE --format "{{.Replicas}}" | cut -d/ -f1)

    if (( $(echo "$CPU > $SCALE_UP_THRESHOLD" | bc -l) )) && [ "$CURRENT" -lt "$MAX_REPLICAS" ]; then
        NEW=$((CURRENT + 1))
        docker service scale $SERVICE=$NEW
        echo "Scaled up to $NEW replicas (CPU: $CPU%)"
    elif (( $(echo "$CPU < $SCALE_DOWN_THRESHOLD" | bc -l) )) && [ "$CURRENT" -gt "$MIN_REPLICAS" ]; then
        NEW=$((CURRENT - 1))
        docker service scale $SERVICE=$NEW
        echo "Scaled down to $NEW replicas (CPU: $CPU%)"
    fi

    sleep 30
done
```

### Option 2: Using Docker Swarm Autoscaler

Deploy the `stefanprodan/swarm-cronjob` or Docker Swarm autoscaler tools:

```yaml
services:
  autoscaler:
    image: stefanprodan/swarm-cronjob
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    deploy:
      placement:
        constraints:
          - node.role == manager
```

## Scaling with Update Parallelism

When scaling up with rolling updates configured, you control how many new replicas start simultaneously:

```bash
# Update service to set parallelism for future updates
docker service update \
  --update-parallelism 2 \
  --update-delay 10s \
  web-frontend
```

## Best Practices

1. **Set resource limits before scaling** — Prevent one service from starving others
2. **Monitor node capacity** — Ensure nodes can handle the additional tasks
3. **Use placement constraints** — Prevent over-loading specific nodes
4. **Configure health checks** — Swarm waits for health checks before marking tasks healthy
5. **Test scale-down** — Verify graceful shutdown handles in-flight requests

## Conclusion

Scaling services in Portainer on Docker Swarm is a straightforward operation that the Swarm orchestrator handles automatically. Whether you scale from the services list, the service detail view, or via CLI, the Swarm manager ensures the desired number of replicas are running across your cluster. For production workloads, complement manual scaling with monitoring and potentially an auto-scaling solution.
