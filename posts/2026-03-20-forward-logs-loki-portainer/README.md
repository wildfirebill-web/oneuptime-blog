# How to Forward Container Logs to Loki via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Loki, Logging, Log Forwarding, Grafana

Description: Forward container logs directly to Grafana Loki using the Loki Docker log driver plugin or Promtail, enabling label-based log querying with LogQL in Portainer environments.

## Introduction

Forwarding logs directly to Loki from Docker containers eliminates the need for a file-based agent polling container log files. There are two approaches: the Loki Docker Driver Plugin (installed on the Docker host) which intercepts logs at the driver level, and Promtail which reads Docker's JSON log files. This guide covers both methods with production configurations for Portainer-managed environments.

## Method 1: Loki Docker Driver Plugin

### Step 1: Install the Loki Docker Plugin

```bash
# Install the Loki Docker driver plugin on each host
docker plugin install grafana/loki-docker-driver:latest \
  --alias loki \
  --grant-all-permissions

# Verify plugin is installed and active
docker plugin ls
# Should show: loki (enabled)

# The plugin intercepts container logs at the kernel level
# No agent container needed with this approach
```

### Step 2: Configure Loki as Default Log Driver

```json
// /etc/docker/daemon.json - Use Loki for all containers
{
  "log-driver": "loki",
  "log-opts": {
    "loki-url": "http://loki.internal.com:3100/loki/api/v1/push",
    "loki-batch-size": "400",
    "loki-retries": "5",
    "loki-max-backoff": "800ms",
    "loki-timeout": "1s",
    "loki-external-labels": "host=docker-host-1,environment=production"
  }
}
```

```bash
sudo systemctl restart docker

# Verify
docker info | grep "Logging Driver"
# Returns: Logging Driver: loki
```

## Method 2: Per-Container Loki Configuration

### Step 3: Configure Loki Logging per Service

```yaml
# docker-compose.yml - Loki logging per service
version: "3.8"

services:
  api:
    image: myapp/api:latest
    logging:
      driver: loki
      options:
        # Loki push API endpoint
        loki-url: "http://loki:3100/loki/api/v1/push"
        # Labels attached to every log line
        loki-labels: "service=api,environment=production,stack=myapp"
        # Performance tuning
        loki-batch-size: "400"          # Bytes per batch
        loki-retries: "5"               # Retry failed sends
        loki-max-backoff: "800ms"       # Max retry backoff
        loki-timeout: "1s"              # Per-request timeout
        # Include Docker container labels as Loki labels
        loki-external-labels: "container_name={{.Name}},image={{.ImageName}}"
        # Pipeline stages for log parsing
        loki-pipeline-stages: |
          - json:
              expressions:
                level: level
                msg: msg
          - labels:
              level:
          - timestamp:
              source: timestamp
              format: RFC3339Nano

  nginx:
    image: nginx:alpine
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-labels: "service=nginx,environment=production"
        # Parse nginx access log format
        loki-pipeline-stages: |
          - regex:
              expression: '^(?P<ip>\S+) - - \[(?P<time>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) \S+" (?P<status>\d+) (?P<bytes>\d+)'
          - labels:
              status:
              method:
```

## Method 3: Promtail for File-Based Collection

### Step 4: Deploy Promtail as Sidecar or Global Service

```yaml
# docker-compose.yml - Promtail reads Docker JSON log files
version: "3.8"

services:
  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    restart: unless-stopped
    command: -config.file=/etc/promtail/config.yaml
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yaml
      # Access Docker socket for container metadata
      - /var/run/docker.sock:/var/run/docker.sock:ro
      # Access Docker log files
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      # Store position tracking
      - promtail_pos:/tmp/promtail
    networks:
      - logging_net

volumes:
  promtail_pos:
```

```yaml
# promtail-config.yaml - Comprehensive Docker log collection
server:
  http_listen_port: 9080

positions:
  filename: /tmp/promtail/positions.yaml
  sync_period: 10s

clients:
  - url: http://loki:3100/loki/api/v1/push
    batchwait: 1s
    batchsize: 1048576
    # Timeout for push requests
    timeout: 10s

scrape_configs:
  - job_name: docker_containers
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s

    relabel_configs:
      - source_labels: ["__meta_docker_container_name"]
        regex: "/(.*)"
        target_label: container
      - source_labels:
          ["__meta_docker_container_label_com_docker_compose_service"]
        target_label: service
      - source_labels:
          ["__meta_docker_container_label_com_docker_compose_project"]
        target_label: stack
      - source_labels: ["__meta_docker_container_log_stream"]
        target_label: stream

    pipeline_stages:
      # Parse JSON logs
      - json:
          expressions:
            level: level
            msg: msg
            error: error
            duration: duration
      # Promote level to label for efficient filtering
      - labels:
          level:
      # Handle timestamp from log content
      - timestamp:
          source: timestamp
          format: RFC3339Nano
          fallback_formats:
            - "2006-01-02T15:04:05Z07:00"
            - "2006-01-02 15:04:05"
```

## Step 5: Query Logs with LogQL

```bash
# Essential LogQL queries for Loki:

# All logs from a service
{service="api"}

# Error logs with label filtering
{service="api", level="error"}

# Search log content (slower - scans content)
{stack="myapp"} |= "database connection"

# Parse JSON and filter
{service="api"} | json | duration > 1000ms

# Extract HTTP status codes
{service="nginx"} | pattern `<ip> - - [<_>] "<method> <path> <_>" <status> <bytes>`
  | status >= 500

# Rate of errors over time (metric query)
sum(rate({stack="myapp", level="error"}[5m])) by (service)

# Top services by log volume
topk(5, sum by (service) (rate({stack="myapp"}[5m])))
```

## Step 6: Verify Logs Are Arriving in Loki

```bash
# Query Loki API directly to verify reception
curl -G -s "http://loki:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={service="api"}' \
  --data-urlencode "start=$(date -d '5 minutes ago' +%s)000000000" \
  --data-urlencode "end=$(date +%s)000000000" \
  --data-urlencode "limit=10" | jq '.data.result[].values[][1]'

# Check Loki ingestion stats
curl -s http://loki:3100/metrics | grep -E "loki_distributor|ingester"

# Verify labels are being applied correctly
curl -s "http://loki:3100/loki/api/v1/labels" | jq '.data[]'
# Should include: service, stack, level, container

curl -s "http://loki:3100/loki/api/v1/label/service/values" | jq '.data[]'
# Should list all your services
```

## Conclusion

The Loki Docker driver plugin is the cleanest approach for new deployments — zero configuration per container, all metadata captured automatically, and no additional agent containers needed. Promtail provides more flexibility for existing environments where changing the log driver isn't feasible, or where you need advanced pipeline stages for complex log parsing. Both methods support LogQL's label-based filtering, which is far more efficient than full-text search for typical operational queries. Portainer's compose YAML manages both Loki infrastructure and the application services that ship logs to it from a single management interface.
