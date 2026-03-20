# How to Configure Container Logging Drivers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Logging, Observability

Description: Learn how to configure Docker logging drivers for containers in Portainer, including json-file, syslog, Fluentd, and Loki integrations.

## Introduction

Docker supports multiple logging drivers that control how container logs are collected, stored, and forwarded. Portainer allows you to configure the logging driver and its options when creating a container. Choosing the right logging driver is critical for observability and operational efficiency.

## Prerequisites

- Portainer installed with a connected Docker environment
- Understanding of your logging infrastructure (e.g., Elasticsearch, Loki, Splunk)

## Available Logging Drivers

| Driver | Description | Best For |
|--------|-------------|----------|
| `json-file` | Store logs as JSON files on host | Default; local development |
| `local` | Compressed binary format on host | Performance-sensitive local logging |
| `syslog` | Send to syslog daemon | Unix/Linux syslog systems |
| `journald` | Send to systemd journal | systemd-based systems |
| `fluentd` | Send to Fluentd aggregator | EFK stack (Elasticsearch, Fluentd, Kibana) |
| `awslogs` | Send to AWS CloudWatch Logs | AWS deployments |
| `gcplogs` | Send to Google Cloud Logging | GCP deployments |
| `splunk` | Send to Splunk HTTP Event Collector | Splunk users |
| `gelf` | Send to GELF endpoint (Graylog) | Graylog users |
| `loki` | Send to Grafana Loki | Loki/Grafana stack |
| `none` | Disable logging | High-throughput, no logging needed |

## Step 1: Configure Logging Driver in Portainer

1. Navigate to **Containers > Add container**.
2. Scroll to the **Logging** tab.
3. Under **Logging driver**, select from the dropdown.
4. Add driver-specific options as key-value pairs.

## Step 2: Configure Common Logging Drivers

### json-file (Default)

```yaml
# docker-compose.yml equivalent

logging:
  driver: json-file
  options:
    max-size: "10m"    # Rotate log files at 10 MB
    max-file: "5"      # Keep 5 rotated log files
    compress: "true"   # Compress rotated files
```

In Portainer logging options:
```text
max-size: 10m
max-file: 5
compress: true
```

### syslog

```yaml
logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.1.50:514"  # Remote syslog server
    syslog-facility: "daemon"
    tag: "myapp/{{.Name}}"  # Tag logs with container name
```

### Fluentd

```yaml
logging:
  driver: fluentd
  options:
    fluentd-address: "localhost:24224"
    fluentd-async: "true"    # Non-blocking; don't fail container if Fluentd is down
    tag: "docker.{{.Name}}"
```

### Loki (via Grafana Loki Docker plugin)

First, install the Loki plugin on the Docker host:

```bash
# Install Grafana Loki logging driver plugin
docker plugin install grafana/loki-docker-driver:latest \
    --alias loki \
    --grant-all-permissions
```

Then configure:

```yaml
logging:
  driver: loki
  options:
    loki-url: "http://loki:3100/loki/api/v1/push"
    loki-batch-size: "400"
    loki-external-labels: "container_name={{.Name}},site=${SITE:-default}"
```

In Portainer logging options:
```text
loki-url: http://loki:3100/loki/api/v1/push
loki-batch-size: 400
loki-external-labels: container_name={{.Name}}
```

### AWS CloudWatch Logs

```yaml
logging:
  driver: awslogs
  options:
    awslogs-region: "us-east-1"
    awslogs-group: "/docker/production"
    awslogs-stream: "{{.Name}}"
    awslogs-create-group: "true"
```

## Step 3: Set a Default Logging Driver for All Containers

Instead of configuring per-container, set a default in Docker daemon:

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5",
    "compress": "true"
  }
}
```

```bash
# Apply the new daemon configuration
systemctl restart docker
```

All new containers will use this driver unless overridden.

## Step 4: Override the Default for Specific Containers

In Portainer, you can override the daemon default for any container:

```text
# Container uses none driver (silences noisy debug containers)
Driver: none

# Or use a different driver for a high-value service
Driver: fluentd
Options:
  fluentd-address: fluentd.internal:24224
  tag: "production.{{.Name}}"
```

## Log Availability in Portainer

Important: **Portainer's built-in log viewer only works with the `json-file` and `journald` drivers**.

If you use `fluentd`, `loki`, `syslog`, or other remote drivers:
- Portainer cannot display logs in the web UI.
- Logs are forwarded to the remote system.
- View logs in the remote system (Grafana, Graylog, etc.).

If you need both Portainer log viewing AND forwarding, use `json-file` locally and a log shipping agent (like Fluent Bit) to forward logs to your central system.

## Best Practices

- **Always set `max-size` and `max-file`** for `json-file` driver to prevent disk exhaustion.
- **Use `fluentd-async: true`** with Fluentd to prevent container startup failures if Fluentd is unavailable.
- **Use centralized logging** for production - json-file on every host doesn't scale.
- **Add context labels** to logs (container name, site, environment) for easier filtering.

## Conclusion

Logging driver configuration in Portainer gives you control over where and how container logs are stored. For simple setups, the default `json-file` driver with size rotation is sufficient. For production environments with many containers, consider centralized logging via Fluentd, Loki, or CloudWatch to aggregate logs from all containers in one searchable place.
