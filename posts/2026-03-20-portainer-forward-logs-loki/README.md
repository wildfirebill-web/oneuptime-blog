# How to Forward Container Logs to Loki via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Loki, Log Forwarding, Docker, Grafana, Centralized Logging

Description: Learn how to forward Docker container logs to Loki via Portainer using the Docker Loki log driver or Promtail for label-rich log ingestion.

---

Grafana Loki stores logs indexed by labels rather than full text, making it highly storage-efficient. This guide covers two methods for forwarding container logs to Loki: the native Docker Loki log driver and Promtail.

## Method 1: Docker Loki Log Driver

The official Loki Docker driver plugin sends logs directly from Docker to Loki without an intermediate agent.

### Install the Loki Plugin

Run on each Docker host:

```bash
docker plugin install grafana/loki-docker-driver:latest \
  --alias loki \
  --grant-all-permissions

# Verify it's installed

docker plugin ls
```

### Configure Containers to Use the Loki Driver

In your Portainer stack:

```yaml
version: "3.8"

services:
  api:
    image: my-api:latest
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-batch-size: "400"
        loki-external-labels: "job=my-app,service=api,environment=production"
        loki-retries: "5"
        loki-min-backoff: "100ms"
        loki-max-backoff: "10s"

  worker:
    image: my-worker:latest
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-external-labels: "job=my-app,service=worker,environment=production"
```

### Set Loki as Default Log Driver

Apply to all containers on the host via `/etc/docker/daemon.json`:

```json
{
  "log-driver": "loki",
  "log-opts": {
    "loki-url": "http://loki-host:3100/loki/api/v1/push",
    "loki-external-labels": "host=my-server,environment=production",
    "loki-batch-size": "400"
  }
}
```

## Method 2: Promtail (Agent-Based)

Promtail reads Docker container log files and enriches them with labels before sending to Loki. This is the recommended approach for auto-discovery of all containers.

### Deploy Promtail as a Stack

```yaml
version: "3.8"

services:
  promtail:
    image: grafana/promtail:2.9.4
    command: -config.file=/etc/promtail/config.yml
    volumes:
      - ./promtail.yaml:/etc/promtail/config.yml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - promtail_positions:/tmp
    networks:
      - logging_net

volumes:
  promtail_positions:

networks:
  logging_net:
    external: true
```

### Promtail Configuration with Docker Service Discovery

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push
    batchwait: 1s
    batchsize: 102400
    timeout: 10s
    backoff_config:
      min_period: 500ms
      max_period: 5m
      max_retries: 10

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    pipeline_stages:
      - docker: {}
      - json:
          expressions:
            output: log
            stream: stream
      - labels:
          stream:
      - output:
          source: output
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: container
      - source_labels: ['__meta_docker_container_image']
        target_label: image
      - source_labels: ['__meta_docker_compose_service']
        target_label: service
      - source_labels: ['__meta_docker_compose_project']
        target_label: stack
      - source_labels: ['__meta_docker_container_label_com_docker_compose_project']
        target_label: stack
```

## Querying in Grafana

After forwarding logs to Loki, query them with LogQL in Grafana's Explore:

```logql
# All logs from a specific stack
{stack="my-app"} | json

# Error-level logs across all services
{environment="production"} |= "level=error"

# Rate of errors per service
sum by (service) (
  rate({stack="my-app"} |= "error" [5m])
)

# Last 100 log lines for a container
{container="my-app_api_1"} | line_format "{{.msg}}"
```

## Comparing the Two Methods

| Feature | Loki Driver | Promtail |
|---------|-------------|----------|
| Setup | Plugin install required | Just deploy a container |
| Auto-discovery | Manual labeling | Automatic via Docker SD |
| Label richness | Manual `loki-external-labels` | Full Docker metadata |
| Failure handling | Plugin handles | Configurable with retries |
| Recommended for | Simple, static setups | Production, multi-service |
