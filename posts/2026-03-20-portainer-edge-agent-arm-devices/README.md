# How to Run Portainer Edge Agent on ARM Devices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge Agent, ARM, Raspberry Pi, IoT, Edge Computing

Description: Deploy the Portainer Edge Agent on ARM-based devices such as Raspberry Pi and other SBCs to bring IoT and edge hardware under centralized Portainer management.

## Introduction

ARM devices — Raspberry Pi, Jetson Nano, Orange Pi, and similar single-board computers — are widely used for IoT, edge computing, and lightweight workloads. Portainer's Edge Agent supports ARM32 (armv7) and ARM64 (aarch64) architectures, allowing these devices to be managed centrally from a Portainer Business Edition server. This guide covers deploying the Edge Agent on ARM hardware with best practices for low-resource and intermittently connected environments.

## Supported Architectures

| Architecture | Example Devices | Docker Image Tag |
|---|---|---|
| ARM 32-bit (armv7) | Raspberry Pi 2/3 (32-bit OS) | `portainer/agent:latest` |
| ARM 64-bit (aarch64) | Raspberry Pi 4/5 (64-bit OS), Jetson Nano | `portainer/agent:latest` |

Portainer's official images are multi-arch manifests — the correct variant is pulled automatically based on the host architecture.

## Prerequisites

- ARM device running Raspberry Pi OS, Ubuntu ARM, or similar Linux distribution
- Docker installed on the ARM device
- Portainer Business Edition server accessible from the device
- Network access from the ARM device to the Portainer server (outbound on ports 8000 and 9443)

## Step 1: Install Docker on the ARM Device

```bash
# On Raspberry Pi OS or Debian/Ubuntu ARM
curl -fsSL https://get.docker.com | sh

# Add the current user to the docker group
sudo usermod -aG docker $USER

# Enable Docker to start on boot
sudo systemctl enable docker
sudo systemctl start docker

# Verify
docker info | grep -E "Architecture|Server Version"
```

## Step 2: Create an Edge Environment in Portainer

In the Portainer UI:

1. Go to **Environments** → **Add environment**
2. Select **Docker Standalone** → **Edge Agent**
3. Set the name (e.g., `rpi-sensor-node-01`)
4. Enter the Portainer server URL: `https://portainer.example.com`
5. Click **Create**
6. Copy the **Edge ID** and **Edge Key**

## Step 3: Deploy the Edge Agent

```bash
# Set variables
EDGE_ID="your-edge-id-here"
EDGE_KEY="your-edge-key-here"
PORTAINER_URL="https://portainer.example.com"

# Pull the multi-arch image (ARM variant downloaded automatically)
docker pull portainer/agent:latest

# Run the Edge Agent
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -e EDGE=1 \
  -e EDGE_ID="${EDGE_ID}" \
  -e EDGE_KEY="${EDGE_KEY}" \
  -e EDGE_INSECURE_POLL=0 \
  --name portainer_edge_agent \
  --restart always \
  portainer/agent:latest
```

## Step 4: Configure for Low-Resource Environments

ARM devices often have limited RAM (512MB–4GB). Tune the agent intervals to reduce resource usage:

```bash
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -e EDGE=1 \
  -e EDGE_ID="${EDGE_ID}" \
  -e EDGE_KEY="${EDGE_KEY}" \
  -e EDGE_PING_INTERVAL=30 \
  -e EDGE_CMD_INTERVAL=30 \
  -e EDGE_SNAPSHOT_INTERVAL=300 \
  --name portainer_edge_agent \
  --restart always \
  --memory="128m" \
  --cpus="0.5" \
  portainer/agent:latest
```

Increasing the intervals from the default 5 seconds to 30 seconds reduces CPU wakeups significantly on devices with microSD storage.

## Step 5: Async Mode for Unreliable Connectivity

IoT and edge devices often have intermittent connectivity (cellular, LoRa backhaul, or WiFi dropouts). Enable async mode:

```bash
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -e EDGE=1 \
  -e EDGE_ASYNC=1 \
  -e EDGE_ID="${EDGE_ID}" \
  -e EDGE_KEY="${EDGE_KEY}" \
  -e EDGE_PING_INTERVAL=120 \
  -e EDGE_CMD_INTERVAL=120 \
  -e EDGE_SNAPSHOT_INTERVAL=600 \
  --name portainer_edge_agent \
  --restart always \
  portainer/agent:latest
```

In async mode, the agent polls on a schedule rather than maintaining a persistent tunnel. Portainer queues commands and the device picks them up on its next poll cycle.

## Step 6: Docker Compose Deployment

Create `/opt/portainer/docker-compose.yml`:

```yaml
version: "3.8"

services:
  portainer_edge_agent:
    image: portainer/agent:latest
    container_name: portainer_edge_agent
    restart: always
    environment:
      EDGE: "1"
      EDGE_ID: "${EDGE_ID}"
      EDGE_KEY: "${EDGE_KEY}"
      EDGE_ASYNC: "1"
      EDGE_PING_INTERVAL: "60"
      EDGE_CMD_INTERVAL: "60"
      EDGE_SNAPSHOT_INTERVAL: "300"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    mem_limit: 128m
    cpus: 0.5
```

Create `/opt/portainer/.env`:

```
EDGE_ID=your-edge-id-here
EDGE_KEY=your-edge-key-here
```

```bash
cd /opt/portainer
docker compose up -d
```

## Step 7: Auto-Start on Boot with systemd

Ensure Docker and the Edge Agent start automatically after a power cycle:

```bash
# Enable Docker on boot
sudo systemctl enable docker

# Create a systemd service for Docker Compose
sudo tee /etc/systemd/system/portainer-edge.service << 'EOF'
[Unit]
Description=Portainer Edge Agent
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/portainer
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable portainer-edge
sudo systemctl start portainer-edge
```

## Step 8: Bulk Provisioning Multiple ARM Devices

For deploying the same Edge Agent configuration to a fleet of ARM devices, use the Portainer auto-onboarding feature with a shared edge key:

```bash
#!/bin/bash
# provision-arm-device.sh
# Run this script on each ARM device during initial setup

PORTAINER_URL="https://portainer.example.com"
EDGE_GROUP_KEY="shared-group-edge-key"  # From Portainer edge group settings

# Generate a unique EDGE_ID per device using hostname + MAC
DEVICE_MAC=$(ip link show eth0 | awk '/ether/ {print $2}' | tr -d ':')
EDGE_ID="arm-${HOSTNAME}-${DEVICE_MAC}"

docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -e EDGE=1 \
  -e EDGE_ID="${EDGE_ID}" \
  -e EDGE_KEY="${EDGE_GROUP_KEY}" \
  -e EDGE_ASYNC=1 \
  -e EDGE_PING_INTERVAL=60 \
  --name portainer_edge_agent \
  --restart always \
  portainer/agent:latest

echo "Edge Agent deployed with ID: ${EDGE_ID}"
echo "Device will appear in Portainer waiting room for approval"
```

New devices appear in the **Waiting Room** under Edge Environments. An administrator approves them (or enable auto-admit) to bring them under management.

## Monitoring ARM Device Health

After connecting ARM devices, use Portainer's environment health overview to monitor across your fleet. Combine with a dedicated monitoring tool to track CPU temperature, memory pressure, and disk space on the ARM devices themselves.

## Troubleshooting

**Wrong architecture image pulled:**
```bash
# Verify the running image architecture
docker inspect portainer_edge_agent | grep Architecture
# Should show "arm" or "arm64"
```

**Agent fails to start on Raspberry Pi OS Lite (32-bit):**
- Ensure the kernel has cgroups v1 or v2 enabled
- Add `cgroup_enable=memory cgroup_memory=1` to `/boot/cmdline.txt` and reboot

**High SD card write rate:**
- Increase snapshot and ping intervals to reduce log writes
- Mount Docker volumes on an external USB SSD rather than the microSD card

**Cannot reach Portainer server:**
```bash
# Test connectivity
curl -sk https://portainer.example.com/api/status
nc -zv portainer.example.com 8000
```

## Conclusion

The Portainer Edge Agent's multi-architecture support makes it a natural fit for ARM-based edge deployments. Whether managing a handful of Raspberry Pi nodes or a fleet of hundreds of IoT devices spread across multiple sites, the Edge Agent's outbound-only communication model and async polling mode handle unreliable connectivity gracefully. Combine auto-onboarding with edge groups and dynamic tags to build a scalable zero-touch provisioning pipeline for ARM device fleets.
