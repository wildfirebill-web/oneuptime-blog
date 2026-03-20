# How to Set Up Centralized Logging for Containers via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Logging, Centralized Logging, Observability, Log Management

Description: Configure centralized log collection from all containers in your Portainer environment using log drivers and log aggregation pipelines.

## Introduction

Docker containers write logs to stdout/stderr by default. Without centralized logging, you need to check each container individually in Portainer. Centralized logging aggregates all container logs into a single system where you can search, filter, and alert across all services simultaneously. This guide covers configuring Docker log drivers and deploying a log aggregation stack via Portainer.

## Step 1: Configure Docker Log Driver Globally

```json
// /etc/docker/daemon.json - Global log driver configuration
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "labels": "service_name,environment",
    "env": "NODE_ENV,APP_VERSION"
  }
}
```

```bash
# Apply and verify
sudo systemctl restart docker
docker info | grep -A 5 "Logging Driver"
```

## Step 2: Deploy the Log Collection Stack

```yaml
# docker-compose.yml - Centralized logging with Loki + Grafana
version: "3.8"

services:
  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    restart: unless-stopped
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
      - loki_data:/loki
    ports:
      - "3100:3100"
    networks:
      - logging_net

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    restart: unless-stopped
    command: -config.file=/etc/promtail/config.yml
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yml
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    networks:
      - logging_net
    depends_on:
      - loki

  grafana:
    image: grafana/grafana:latest
    container_name: grafana_logging
    restart: unless-stopped
    volumes:
      - grafana_logging_data:/var/lib/grafana
      - ./grafana-datasources.yaml:/etc/grafana/provisioning/datasources/loki.yaml
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    networks:
      - logging_net
    depends_on:
      - loki

volumes:
  loki_data:
  grafana_logging_data:

networks:
  logging_net:
    driver: bridge
```

## Step 3: Configure Loki and Promtail

```yaml
# loki-config.yaml - Loki storage configuration
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 1h
  max_chunk_age: 1h

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

limits_config:
  retention_period: 30d  # Keep logs for 30 days
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
```

```yaml
# promtail-config.yaml - Collect from all Docker containers
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker_logs
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      # Use container name as job label
      - source_labels: ["__meta_docker_container_name"]
        regex: "/(.*)"
        target_label: container
      # Use image as service label
      - source_labels: ["__meta_docker_container_log_stream"]
        target_label: stream
      # Include compose service name
      - source_labels:
          ["__meta_docker_container_label_com_docker_compose_service"]
        target_label: service
      # Include stack name
      - source_labels:
          ["__meta_docker_container_label_com_docker_compose_project"]
        target_label: stack
```

## Step 4: Configure Grafana Loki Datasource

```yaml
# grafana-datasources.yaml - Auto-provision Loki datasource
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    isDefault: true
    version: 1
    editable: false
    jsonData:
      maxLines: 1000
```

## Step 5: Query Logs with LogQL

```bash
# LogQL queries for common use cases in Grafana Explore:

# All logs from a specific container
{container="api"}

# Error logs across all containers
{stack="myapp"} |= "error" | json

# Logs from a specific stack
{stack="myapp"} | json | line_format "{{.container}}: {{.msg}}"

# Count errors per container over 5 minutes
sum by (container) (
  count_over_time({stack="myapp"} |= "error" [5m])
)

# Extract HTTP status codes
{container="nginx"} | pattern '<ip> - - [<_>] "<method> <path> <_>" <status> <_>'
| status >= 500

# Tail logs from Portainer CLI equivalent
# curl -G -s "http://localhost:3100/loki/api/v1/query_range" \
#   --data-urlencode 'query={container="api"}' \
#   --data-urlencode "start=$(date -d '5 minutes ago' +%s)000000000" \
#   --data-urlencode "end=$(date +%s)000000000" | jq
```

## Step 6: Send Container Logs via Docker Log Driver to Loki

```yaml
# Configure containers to send directly to Loki
version: "3.8"

services:
  api:
    image: myapp/api:latest
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-labels: "service=api,environment=production"
        loki-retries: 5
        loki-batch-size: 400
        loki-external-labels: "host=${HOSTNAME}"
```

## Conclusion

Centralized logging transforms troubleshooting from container-by-container log inspection to cross-service search and correlation. The PLG stack (Promtail + Loki + Grafana) is resource-efficient — Loki indexes only log metadata (labels), not the log content, keeping storage costs low. Promtail's Docker service discovery automatically picks up new containers without configuration changes. Portainer's log viewer remains useful for quick checks, while Grafana Explore and Loki's LogQL handle complex cross-service queries, error aggregation, and alerting.
