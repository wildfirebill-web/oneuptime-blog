# How to Set Up Portainer for Industrial IoT Edge Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Edge Computing, IoT, Industrial

Description: Learn how to configure Portainer for managing containerized workloads on industrial IoT edge devices including PLCs, gateways, and HMI systems.

## Introduction

Industrial IoT (IIoT) environments present unique challenges: constrained hardware, unreliable connectivity, strict uptime requirements, and the need to run OT (Operational Technology) protocols alongside modern containerized apps. Portainer's Edge Compute capabilities are well-suited for managing these environments from a central control plane.

## Prerequisites

- Portainer Business Edition (BE) installed on a central server
- Linux-based edge devices (e.g., industrial PCs, Raspberry Pi 4, NVIDIA Jetson, or x86 gateways)
- Docker Engine installed on edge devices
- Network connectivity (even intermittent) between edge devices and Portainer server

## IIoT Edge Architecture Overview

A typical IIoT edge setup with Portainer looks like this:

- **Portainer Server**: Runs in your data center or cloud.
- **Edge Gateways**: Industrial PCs running Docker + Portainer Edge Agent.
- **Containers on Gateways**:
  - OPC-UA server/client
  - MQTT broker
  - Data preprocessing containers
  - Time-series database (InfluxDB, TimescaleDB)
  - Visualization (Grafana)

## Step 1: Install Docker on Edge Hardware

For Debian/Ubuntu-based industrial Linux:

```bash
#!/bin/bash
# Install Docker Engine on an industrial Linux gateway

# Update package index
apt-get update

# Install prerequisites
apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io

# Enable and start Docker
systemctl enable docker
systemctl start docker
```

## Step 2: Deploy the Portainer Edge Agent

In Portainer, create a new Edge endpoint and copy the generated Edge Key. Then on each device:

```bash
#!/bin/bash
# Deploy Portainer Edge Agent on IIoT gateway
# Designed for intermittent connectivity environments

EDGE_KEY="your-edge-key-from-portainer"
DEVICE_NAME="factory-gateway-berlin-01"
SITE="berlin-factory"
ROLE="gateway"

docker run -d \
  --name portainer_edge_agent \
  --restart always \
  # Mount Docker socket for container management
  -v /var/run/docker.sock:/var/run/docker.sock \
  # Mount Docker volumes for volume browsing
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  # Mount host root for node inspection
  -v /:/host \
  -e EDGE_KEY="${EDGE_KEY}" \
  # Increase poll frequency for bandwidth-constrained links
  -e EDGE_POLL_FREQUENCY=60 \
  # Tags for dynamic group assignment
  -e EDGE_TAGS="site=${SITE},role=${ROLE},device=${DEVICE_NAME}" \
  portainer/agent:latest
```

## Step 3: Deploy a Standard IIoT Edge Stack

Create this as an Edge Stack in Portainer and deploy it to your `IIoT-Gateways` group:

```yaml
# iiot-gateway-stack.yml
# Standard stack for industrial IoT edge gateways
version: "3.8"

services:
  # MQTT broker for device communication
  mosquitto:
    image: eclipse-mosquitto:2.0
    restart: always
    ports:
      - "1883:1883"   # MQTT
      - "8883:8883"   # MQTT over TLS
    volumes:
      - mosquitto_data:/mosquitto/data
      - mosquitto_logs:/mosquitto/log
      - /etc/edge-configs/mosquitto.conf:/mosquitto/config/mosquitto.conf:ro

  # OPC-UA server for PLC integration
  opcua-server:
    image: mcr.microsoft.com/iotedge/opc-publisher:latest
    restart: always
    environment:
      - PCS_IOTHUB_CONNSTRING=${IOT_HUB_CONNECTION_STRING}
    volumes:
      - /etc/edge-configs/pn.json:/appdata/pn.json:ro

  # Time-series database for local data storage
  influxdb:
    image: influxdb:2.7-alpine
    restart: always
    ports:
      - "8086:8086"
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=${INFLUXDB_PASSWORD}
      - DOCKER_INFLUXDB_INIT_ORG=myorg
      - DOCKER_INFLUXDB_INIT_BUCKET=sensors
    volumes:
      - influxdb_data:/var/lib/influxdb2

  # Visualization dashboard
  grafana:
    image: grafana/grafana:latest
    restart: always
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_PATHS_DATA=/var/lib/grafana
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - influxdb

volumes:
  mosquitto_data:
  mosquitto_logs:
  influxdb_data:
  grafana_data:
```

## Step 4: Handle Device Permissions for Hardware Access

Industrial devices often need container access to serial ports, USB devices, or GPIOs:

```yaml
# Add to your service definition for hardware access
services:
  plc-connector:
    image: myorg/plc-connector:1.0
    restart: always
    # Map serial ports for PLC communication
    devices:
      - /dev/ttyS0:/dev/ttyS0   # RS-232 serial port
      - /dev/ttyUSB0:/dev/ttyUSB0  # USB-serial adapter
    # Required for accessing /dev devices
    privileged: false
    cap_add:
      - SYS_RAWIO
    group_add:
      - dialout  # Add container user to dialout group for serial access
```

## Step 5: Configure Resilient Operation

IIoT devices must continue operating even when disconnected from Portainer:

```yaml
# All containers should use restart: always
# Also configure healthchecks for automatic recovery

services:
  data-collector:
    image: myorg/collector:2.0
    restart: always
    healthcheck:
      # Verify the collector is publishing data
      test: ["CMD", "curl", "-f", "http://localhost:9090/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s
```

## Best Practices for IIoT Edge

- **Design for offline-first**: Every container should work without cloud connectivity.
- **Use local persistence**: Store time-series data locally with forward sync.
- **Secure device access**: Use TLS for MQTT, restrict container capabilities.
- **Monitor resource usage**: Industrial gateways often have limited RAM (2-4 GB).
- **Stage deployments**: Test firmware/stack updates on a test gateway before fleet rollout.

## Conclusion

Portainer provides a reliable management layer for IIoT edge deployments. By combining Docker's containerization benefits with Portainer's central management, you can deploy, update, and monitor industrial workloads — OPC-UA servers, MQTT brokers, time-series databases — across your entire factory floor from a single pane of glass.
