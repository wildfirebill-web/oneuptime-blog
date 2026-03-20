# How to Set Up Node Exporter for Host Metrics with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Node Exporter, Prometheus, Host Metrics, Monitoring

Description: Learn how to deploy Prometheus Node Exporter as a Portainer stack to collect hardware and operating system metrics from your Docker host servers.

## What Is Node Exporter?

Node Exporter is a Prometheus exporter for hardware and OS metrics on Linux hosts. While cAdvisor monitors containers, Node Exporter monitors the host itself — CPU, memory, disk, network, filesystem, and more.

## Metrics Node Exporter Collects

- CPU usage (per-core, idle, iowait, steal)
- Memory (available, used, swap, buffers, cache)
- Disk usage (per-partition usage, I/O rates)
- Network (bytes/packets per interface, errors)
- Filesystem (inodes, bytes used)
- System (load average, uptime, open file descriptors)

## Deploy Node Exporter via Portainer

**Stacks → Add Stack → node-exporter**

```yaml
version: "3.8"

services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    pid: host              # Required to get accurate host metrics
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
      # Enable additional collectors
      - '--collector.systemd'
      - '--collector.processes'
    ports:
      - "9100:9100"
    networks:
      - monitoring

networks:
  monitoring:
    name: monitoring
    external: true
```

Note: `pid: host` allows Node Exporter to accurately measure process metrics.

## Verify Node Exporter

```bash
# Check metrics endpoint
curl http://localhost:9100/metrics | head -20

# Check specific metrics
curl -s http://localhost:9100/metrics | grep "^node_cpu_seconds_total"
curl -s http://localhost:9100/metrics | grep "^node_memory_MemAvailable_bytes"
curl -s http://localhost:9100/metrics | grep "^node_disk_io_time_seconds_total"
```

## Configure Prometheus to Scrape Node Exporter

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'node-exporter'
    scrape_interval: 15s
    static_configs:
      - targets: ['node-exporter:9100']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+)(:[0-9]+)?'
        replacement: '${1}'
```

## Key Metrics for Alerting

```yaml
# prometheus-alerts.yml
groups:
  - name: node-alerts
    rules:
      # High CPU load
      - alert: HighCPULoad
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m

      # Low disk space
      - alert: LowDiskSpace
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 15
        for: 5m

      # High memory usage
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
```

## Grafana Dashboard

Import dashboard ID `1860` (Node Exporter Full). This community dashboard provides:
- CPU usage breakdown
- Memory usage timeline
- Disk I/O rates
- Network bandwidth per interface
- System load

## Multi-Host Setup

For multiple servers, deploy Node Exporter on each host and update Prometheus:

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets:
          - 'server1:9100'
          - 'server2:9100'
          - 'server3:9100'
        labels:
          environment: production
```

Or use Prometheus service discovery for dynamic host lists.

## Conclusion

Node Exporter deployed via Portainer bridges the gap between container-level metrics (from cAdvisor) and host-level metrics. Together they provide full-stack visibility: from the kernel's CPU scheduler and disk I/O all the way up to individual container resource consumption.
