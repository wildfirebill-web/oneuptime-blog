# How to Monitor Microservice Logs Across Containers in Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Logging, Microservices, Docker, Loki, Monitoring, Observability

Description: Learn how to monitor and aggregate logs from multiple microservice containers in Portainer, using both the built-in log viewer and centralized logging solutions.

---

When running multiple microservices, logs are scattered across containers. Portainer provides a basic per-container log viewer, and this guide shows how to aggregate logs from all services for cross-service tracing and debugging.

## Viewing Logs per Container in Portainer

For a quick look at a single service:

1. Go to **Containers** in Portainer.
2. Click a container name.
3. Select the **Logs** tab.
4. Enable **Auto-refresh** and **Timestamps**.

## Viewing Stack Logs in Portainer

For all containers in a stack:

1. Go to **Stacks**.
2. Click the stack name.
3. Each service in the stack shows its container status and a link to logs.

## Aggregating Logs with Loki and Grafana

For cross-service log queries, deploy the PLG stack (Promtail + Loki + Grafana):

```yaml
version: "3.8"
services:
  loki:
    image: grafana/loki:latest
    restart: unless-stopped
    ports:
      - "3100:3100"
    volumes:
      - loki_data:/loki

  promtail:
    image: grafana/promtail:latest
    restart: unless-stopped
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./promtail-config.yml:/etc/promtail/config.yml:ro

  grafana:
    image: grafana/grafana:latest
    restart: unless-stopped
    ports:
      - "3200:3000"
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  loki_data:
  grafana_data:
```

## Promtail Configuration

```yaml
# promtail-config.yml

server:
  http_listen_port: 9080

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker-containers
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
    relabel_configs:
      - source_labels: [__meta_docker_container_name]
        target_label: container
      - source_labels: [__meta_docker_container_label_com_docker_compose_service]
        target_label: service
```

## Querying Cross-Service Logs in Grafana

After setup, query logs across all microservices:

```logql
# All errors from any container
{container=~".+"} |= "error"

# Trace a specific request ID across services
{job="docker-containers"} |= "req-12345"

# User service errors only
{service="user-service"} | json | level="error"
```

## Using Portainer for Log Forwarding

Configure each container's log driver to send to the centralized stack:

```yaml
services:
  user-service:
    image: user-service:latest
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        labels: "service,version"
```
