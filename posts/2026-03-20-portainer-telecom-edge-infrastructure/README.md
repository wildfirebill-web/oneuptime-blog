# How to Set Up Portainer for Telecommunications Edge Infrastructure (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Telecommunications, Edge Computing, Portainer, Docker, 5G, NFV, Network Function

Description: Deploy and manage containerized network functions and edge applications at telecom sites using Portainer to simplify NFV workload lifecycle management.

---

Telecommunications operators run distributed infrastructure across thousands of edge sites - central offices, cell towers, and data centers. Portainer's Edge Agent and Kubernetes integration support containerized network functions (CNFs) and edge applications at these sites.

## Telecom Edge Use Cases for Portainer

- **vCPE (Virtual Customer Premises Equipment)** - SD-WAN, firewall, and routing functions
- **MEC (Multi-Access Edge Computing)** - application hosting close to the radio
- **OSS/BSS microservices** - operations and business support systems
- **Network probes** - traffic monitoring and quality measurement

## Step 1: Deploy a vCPE Stack

```yaml
# vcpe-stack.yml - virtual CPE for enterprise customer edge

version: "3.8"

services:
  # Software-defined WAN router
  sd-wan-agent:
    image: telecom/sdwan-agent:6.1.2
    cap_add:
      - NET_ADMIN    # Required for network routing functions
    network_mode: host   # Host networking for full packet access
    environment:
      - CONTROLLER_URL=https://sdwan-controller.telecom.example.com
      - SITE_ID=${SITE_ID}
      - WAN_INTERFACE=eth0
      - LAN_INTERFACE=eth1
    restart: always
    volumes:
      - sdwan-config:/etc/sdwan
      - sdwan-logs:/var/log/sdwan

  # Firewall/UTM function
  firewall:
    image: telecom/ngfw-container:3.4.1
    cap_add:
      - NET_ADMIN
      - NET_RAW
    environment:
      - POLICY_SERVER=https://policy.telecom.example.com
      - SITE_ID=${SITE_ID}
    volumes:
      - firewall-config:/etc/firewall
      - firewall-logs:/var/log/firewall
    restart: always

  # Network quality probe
  quality-probe:
    image: telecom/network-probe:2.0.1
    environment:
      - MEASUREMENT_INTERVAL=30
      - REPORTING_URL=https://nqm.telecom.example.com
      - SITE_ID=${SITE_ID}
    restart: unless-stopped

volumes:
  sdwan-config:
  sdwan-logs:
  firewall-config:
  firewall-logs:
```

## Step 2: Deploy MEC Applications

Multi-access edge computing applications run at the edge of the radio network for ultra-low latency:

```yaml
# mec-stack.yml
version: "3.8"

services:
  # AR/VR content cache and rendering offload
  mec-ar-cache:
    image: mec-platform/ar-content-cache:1.2.0
    environment:
      - CENTRAL_CACHE_URL=https://cdn.telecom.example.com
      - LOCAL_CACHE_SIZE_GB=50
      - SITE_ID=${CELL_SITE_ID}
    volumes:
      - ar-cache:/var/cache/ar-content
    ports:
      - "8443:8443"
    restart: unless-stopped

  # Vehicle-to-everything (V2X) processing
  v2x-processor:
    image: mec-platform/v2x-processor:2.1.0
    environment:
      - RSU_INTERFACE=eth1
      - LATENCY_SLA_MS=10
    cap_add:
      - NET_RAW
    restart: always

volumes:
  ar-cache:
```

## Step 3: Configure High Availability

For critical telecom functions, configure service restart policies and health checks:

```yaml
services:
  sd-wan-agent:
    image: telecom/sdwan-agent:6.1.2
    restart: always
    healthcheck:
      # Check that the SD-WAN agent is connected to its controller
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
    deploy:
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 10
```

## Step 4: Monitor Network Function Health

Deploy a lightweight monitoring stack alongside network functions:

```yaml
  # Prometheus for NFV metrics
  prometheus:
    image: prom/prometheus:v2.50.0
    volumes:
      - /opt/telecom/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    restart: unless-stopped

  # Node exporter for host metrics
  node-exporter:
    image: prom/node-exporter:v1.7.0
    pid: host
    network_mode: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
    restart: unless-stopped
```

## Compliance and Hardening

Telecom operators must meet strict compliance requirements:

- Use signed and verified container images from a private registry
- Enforce container security contexts (no-new-privileges, read-only root fs)
- Log all container operations to a centralized SIEM
- Segment network functions using isolated Docker networks

## Summary

Portainer provides telecom operators with a practical approach to managing containerized network functions at edge sites. Its Edge Agent model - outbound connections only, no inbound firewall rules needed - is well-suited to the security requirements of telecom infrastructure.
