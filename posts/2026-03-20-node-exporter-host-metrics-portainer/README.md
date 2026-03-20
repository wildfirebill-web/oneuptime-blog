# How to Deploy Node Exporter for Host Metrics with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Prometheus, Node Exporter, Monitoring, Linux Metrics

Description: Learn how to deploy Prometheus Node Exporter as a container through Portainer to collect and expose host system metrics including CPU, memory, disk, and network.

## Introduction

Prometheus Node Exporter exposes hardware and OS metrics from Linux hosts. Deploying it via Portainer makes it easy to manage alongside your other containerized services, and the metrics can be scraped by Prometheus for storage and visualization in Grafana.

## Prerequisites

- Portainer installed and running
- Prometheus installed (in a container or on the host)
- A server to monitor

## Step 1: Deploy Node Exporter via Portainer

In Portainer, navigate to **Stacks** > **Add stack** and create a stack named `monitoring`:

```yaml
version: "3.8"

services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    pid: host
    network_mode: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--path.rootfs=/rootfs"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
      - "--collector.netstat"
      - "--collector.meminfo"
      - "--collector.diskstats"
      - "--collector.loadavg"
```

Click **Deploy the stack**.

## Step 2: Verify Node Exporter is Running

In Portainer, go to **Containers** and verify `node-exporter` shows as running. Then test the metrics endpoint:

```bash
curl http://localhost:9100/metrics | head -50
```

You should see output like:

```
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
node_cpu_seconds_total{cpu="0",mode="system"} 234.56
```

## Step 3: Configure Prometheus to Scrape Node Exporter

Add the Node Exporter target to your Prometheus configuration:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
    scrape_interval: 15s
```

If Prometheus runs in a container, use the host IP:

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['host.docker.internal:9100']  # Docker Desktop
        # or the host's internal IP
```

Reload Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload
```

## Step 4: Key Metrics Exposed by Node Exporter

| Metric | Description |
|---|---|
| `node_cpu_seconds_total` | CPU time by mode (idle, user, system) |
| `node_memory_MemTotal_bytes` | Total physical memory |
| `node_memory_MemAvailable_bytes` | Available memory |
| `node_disk_read_bytes_total` | Disk bytes read |
| `node_disk_written_bytes_total` | Disk bytes written |
| `node_network_receive_bytes_total` | Network bytes received per interface |
| `node_network_transmit_bytes_total` | Network bytes transmitted per interface |
| `node_filesystem_avail_bytes` | Available filesystem space |
| `node_load1` | 1-minute load average |
| `node_load5` | 5-minute load average |

## Step 5: Useful PromQL Queries

### CPU Usage Percentage

```promql
100 - (avg by (instance) (
  irate(node_cpu_seconds_total{mode="idle"}[5m])
) * 100)
```

### Memory Usage Percentage

```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

### Disk Usage Percentage

```promql
(1 - node_filesystem_avail_bytes{fstype!="tmpfs"} /
     node_filesystem_size_bytes{fstype!="tmpfs"}) * 100
```

### Network Traffic Rate

```promql
irate(node_network_receive_bytes_total{device="eth0"}[5m])
```

## Step 6: Import a Grafana Dashboard

1. In Grafana, go to **Dashboards** > **Import**.
2. Enter dashboard ID **1860** (Node Exporter Full by rfmoz).
3. Select your Prometheus data source.
4. Click **Import**.

This provides a comprehensive pre-built dashboard covering all major host metrics.

## Securing the Metrics Endpoint

Restrict Node Exporter access to Prometheus only using a firewall:

```bash
# Allow Prometheus to scrape
ufw allow from <prometheus-ip> to any port 9100

# Deny all other access to port 9100
ufw deny 9100
```

Or use Nginx as a proxy with basic authentication in front of Node Exporter.

## Best Practices

- Run Node Exporter with `--collector.disable-defaults` and enable only needed collectors for lower overhead.
- Never expose port 9100 to the public internet.
- Add the `instance` label in Prometheus relabeling to identify hosts by their name.
- Set alerts on high disk usage, memory exhaustion, and high load averages.
- Use Portainer's stack update feature to upgrade Node Exporter versions.

## Conclusion

Deploying Node Exporter through Portainer makes host metrics collection as simple as deploying any other container. Combined with Prometheus and Grafana, you gain comprehensive visibility into CPU, memory, disk, and network performance across all monitored hosts.
