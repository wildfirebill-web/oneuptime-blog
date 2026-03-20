# How to Set Up Netdata via Portainer for Real-Time Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Netdata, Monitoring, Real-Time, Self-Hosted

Description: Deploy Netdata through Portainer to gain per-second real-time visibility into CPU, memory, disk, network, and container metrics on your Docker host.

## Introduction

Netdata provides zero-configuration, per-second granularity monitoring for systems and containers. Unlike Prometheus+Grafana which requires significant configuration, Netdata auto-discovers metrics from hundreds of sources out of the box. Deploying it via Portainer takes under five minutes.

## Prerequisites

- Docker host with Portainer installed
- Port 19999 available

## Deploying Netdata via Portainer Stack

### Step 1: Create the Stack

Navigate to **Stacks > Add stack**, name it `netdata`, and paste:

```yaml
version: "3.8"

services:
  netdata:
    image: netdata/netdata:latest
    container_name: netdata
    hostname: docker-host  # Shown in the Netdata dashboard
    pid: host              # Required for full process monitoring
    network_mode: host     # Needed for accurate network metrics
    cap_add:
      - SYS_PTRACE         # Required for apps plugin
      - SYS_ADMIN          # Required for eBPF plugin
    security_opt:
      - apparmor:unconfined
    volumes:
      # Netdata configuration and persistent data
      - netdataconfig:/etc/netdata
      - netdatalib:/var/lib/netdata
      - netdatacache:/var/cache/netdata
      # Read-only host filesystem mounts for metrics collection
      - /:/host/root:ro,rslave
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /etc/localtime:/etc/localtime:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/os-release:/host/etc/os-release:ro
      # Docker socket for container monitoring
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: unless-stopped
    environment:
      # Disable anonymous statistics reporting
      DO_NOT_TRACK: 1

volumes:
  netdataconfig:
    driver: local
  netdatalib:
    driver: local
  netdatacache:
    driver: local
```

### Step 2: Deploy the Stack

Click **Deploy the stack**. Within 30 seconds the container starts collecting metrics.

Access the dashboard at `http://<your-host>:19999`.

### Step 3: Explore the Dashboard

Netdata auto-discovers and displays:

- **System Overview** - CPU, load, memory, swap, disk I/O, network
- **Containers** - per-container CPU, memory, network, disk I/O (via cgroups)
- **Applications** - per-application grouping with netdata apps plugin
- **Disks** - per-disk throughput, utilization, latency
- **Network Interfaces** - per-interface bandwidth and errors
- **Docker** - container states, image counts

### Step 4: Connect to Netdata Cloud (Optional)

Netdata Cloud provides a free centralized view across multiple nodes:

```bash
# Get the claim token from app.netdata.cloud

# Then claim the node from within the container
docker exec -it netdata netdata-claim.sh \
  -token=<YOUR_CLAIM_TOKEN> \
  -rooms=<YOUR_ROOM_ID> \
  -url=https://app.netdata.cloud
```

Alternatively, add to the stack environment:

```yaml
environment:
  DO_NOT_TRACK: 1
  NETDATA_CLAIM_TOKEN: your_token_here
  NETDATA_CLAIM_URL: https://app.netdata.cloud
  NETDATA_CLAIM_ROOMS: your_room_id
```

### Step 5: Configure Alerts

Netdata ships with pre-configured health checks. To customize, exec into the container:

```bash
# Access Netdata configuration
docker exec -it netdata bash

# Edit health configuration
cd /etc/netdata
./edit-config health.d/cpu.conf
```

Example custom alert:

```conf
# Alert when CPU usage exceeds 80% for 5 minutes
alarm: cpu_usage_high
   on: system.cpu
class: Utilization
 type: System
component: CPU
   os: linux
hosts: *
lookup: average -5m unaligned of user,system,softirq,irq,guest
 units: %
  warn: $this > 80
  crit: $this > 95
 delay: down 15m multiplier 1.5 max 1h
 info: CPU utilization is high
   to: sysadmin
```

### Step 6: Set Up Email Notifications

Edit the notification configuration:

```bash
docker exec -it netdata bash
./edit-config health_alarm_notify.conf
```

Configure SMTP:

```bash
# In health_alarm_notify.conf
SEND_EMAIL="YES"
EMAIL_SENDER="netdata@yourdomain.com"
DEFAULT_RECIPIENT_EMAIL="admin@yourdomain.com"
SMTP_SERVER="smtp.yourdomain.com"
SMTP_PORT=587
SMTP_USER="netdata@yourdomain.com"
SMTP_PASSWORD="your_smtp_password"
SMTP_STARTTLS="YES"
```

## Monitoring Multiple Hosts

Deploy the Netdata agent on each host and connect them all to Netdata Cloud for a unified view, or use Netdata's built-in streaming to a parent node:

```yaml
# On child nodes, add this to environment
environment:
  NETDATA_STREAM_DESTINATION: parent-ip:19999
  NETDATA_STREAM_API_KEY: your-api-key-uuid
```

## Conclusion

Netdata via Portainer delivers immediate, per-second real-time monitoring with zero manual metric configuration. Its auto-discovery engine handles hundreds of data sources automatically, making it ideal for quick deployment in home labs and production environments alike. Connecting to Netdata Cloud gives you centralized visibility across your entire infrastructure.
