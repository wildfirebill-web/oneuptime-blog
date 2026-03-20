# How to View Service Logs in Portainer on Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Logging, Services, DevOps

Description: Learn how to view, filter, and aggregate logs from Docker Swarm services using Portainer's log viewer.

## Introduction

Viewing logs from Swarm services is different from standalone containers because log output comes from multiple replicas running on different nodes. Portainer aggregates these logs and presents them in a unified view, making it much easier to troubleshoot issues across distributed service replicas. This guide covers all methods for accessing and working with Swarm service logs in Portainer.

## Prerequisites

- Portainer installed on Docker Swarm
- At least one service with running tasks
- Admin or operator access

## Step 1: Access Service Logs via Portainer

### From the Services List

1. Navigate to **Services** in the Portainer sidebar
2. Find your service
3. Click the **Logs** icon (document icon) in the row

### From the Service Detail View

1. Click on the service name to open details
2. Click the **Logs** tab at the top of the detail page

## Step 2: Understand the Log Viewer

The Portainer service log viewer aggregates logs from all running tasks:

```
[2024-01-15T10:00:01Z] [web.1] 192.168.1.1 - - [15/Jan/2024] "GET / HTTP/1.1" 200
[2024-01-15T10:00:01Z] [web.2] 192.168.1.2 - - [15/Jan/2024] "GET /api HTTP/1.1" 200
[2024-01-15T10:00:02Z] [web.3] 192.168.1.3 - - [15/Jan/2024] "GET /health HTTP/1.1" 200
```

Each log line is prefixed with:
- Timestamp
- Task name (service.replica-number)

## Step 3: Log Viewer Options

Configure the log display:

```
[x] Auto-scroll     — Jump to newest logs automatically
[x] Wrap lines      — Wrap long lines in viewport
[ ] Timestamps      — Show ISO timestamps per line
Lines: [100 ▼]      — Number of lines to fetch (50, 100, 500, 1000, All)
```

## Step 4: Filter Logs by Task (Replica)

To see logs from a specific replica:

1. From the **Tasks** section, click on a specific task
2. Navigate to that container's logs
3. This shows logs for only that replica

Or from the CLI:

```bash
# Aggregate logs from all replicas
docker service logs web-frontend

# Follow logs in real time
docker service logs -f web-frontend

# Show last 50 lines from all replicas
docker service logs --tail 50 web-frontend

# Show logs with timestamps
docker service logs --timestamps web-frontend

# Filter to a specific task/replica
docker service logs web-frontend.1

# Show logs with raw format (no service prefix)
docker service logs --raw web-frontend
```

## Step 5: Configure Log Drivers for Production

For production Swarm services, configure a centralized log driver:

### JSON File with Rotation (Default)

```yaml
services:
  web:
    image: nginx:alpine
    logging:
      driver: json-file
      options:
        max-size: "10m"    # Rotate after 10MB
        max-file: "5"      # Keep 5 rotated files
```

### Fluentd Log Aggregation

```yaml
services:
  web:
    image: nginx:alpine
    logging:
      driver: fluentd
      options:
        fluentd-address: "fluentd:24224"
        tag: "swarm.{{.Name}}.{{.ID}}"    # Include service name and task ID
        fluentd-async: "true"             # Don't block if Fluentd is unavailable
```

### Loki Log Aggregation

```yaml
services:
  web:
    image: nginx:alpine
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-external-labels: |
          service={{.Name}},
          node={{.NodeName}},
          swarm_stack=my-stack
```

## Step 6: Deploy a Centralized Logging Stack

Deploy Loki + Grafana alongside your application for centralized log management:

```yaml
# logging-stack.yml
version: "3.8"

services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - loki-data:/loki
    deploy:
      placement:
        constraints:
          - node.role == manager

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana-data:/var/lib/grafana
    deploy:
      placement:
        constraints:
          - node.role == manager

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    deploy:
      mode: global    # Run on every node to collect local logs
    command: -config.file=/etc/promtail/config.yml

volumes:
  loki-data:
  grafana-data:
```

## Log Analysis Tips

```bash
# Count error lines across all replicas
docker service logs web-frontend 2>&1 | grep -c "ERROR"

# Find the most recent errors
docker service logs --tail 1000 web-frontend 2>&1 | grep "ERROR" | tail -20

# Monitor a specific pattern in real time
docker service logs -f web-frontend 2>&1 | grep "5[0-9][0-9] "

# Export logs to a file for analysis
docker service logs --timestamps web-frontend > service-logs-$(date +%Y%m%d).log
```

## Conclusion

Portainer's unified log view for Swarm services saves significant time when debugging distributed applications. For development and small deployments, the built-in log viewer is sufficient. For production environments with multiple services, implement a centralized logging solution like Grafana Loki or ELK Stack to retain, search, and alert on logs across your entire Swarm cluster.
