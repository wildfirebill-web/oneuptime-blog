# How to Set Up Automatic Edge Environment Onboarding in Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge Agent, Auto-Onboarding, IoT, Automation, Business Edition

Description: Configure automatic onboarding of Edge environments in Portainer Business Edition to streamline deployment of edge devices at scale.

## Introduction

Automatic edge onboarding allows you to pre-configure edge agents and have them self-register with Portainer without manual intervention for each device. This is essential for large-scale edge deployments with dozens or hundreds of devices.

## Prerequisites

- Portainer Business Edition
- Edge computing features enabled
- Network access from devices to Portainer

## Step 1: Enable Automatic Edge Onboarding

1. Log in to Portainer Business Edition
2. Go to **Settings** → **Edge Compute**
3. Enable **Automatically create edge environments on agent connection**
4. Configure the default group for auto-created environments
5. Configure default tags (optional)
6. Save settings

## Step 2: Create a Pre-Staged Edge Key

Generate an edge key that multiple agents can use:

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Create an edge key for auto-onboarding

curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/edge/keys \
  -d '{
    "name": "factory-default-key",
    "allowAutoOnboarding": true
  }'
```

## Step 3: Pre-Stage Agent Deployment

Create a deployment script that all devices run during provisioning:

```bash
#!/bin/bash
# /opt/provision/install-portainer-agent.sh
# Run this during device initial setup/imaging

# Generate a unique device ID (use hardware identifier if possible)
DEVICE_ID=$(cat /proc/cpuinfo | grep Serial | awk '{print $3}' 2>/dev/null || hostname)
DEVICE_ID=$(echo $DEVICE_ID | tr -d ' :')

# Portainer connection details (embed during imaging)
PORTAINER_URL="https://portainer.example.com"
EDGE_KEY="pre-staged-edge-key"

echo "Installing Portainer Edge Agent for device: $DEVICE_ID"

docker run -d \
  --name portainer_edge_agent \
  --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /var/run/portainer:/var/run/portainer \
  -e EDGE=1 \
  -e EDGE_ID="$DEVICE_ID" \
  -e EDGE_KEY="$EDGE_KEY" \
  -e EDGE_INSECURE_POLL=0 \
  portainer/agent:latest

echo "Edge agent installed. Device will appear in Portainer as: $DEVICE_ID"
```

## Step 4: The Waiting Room

When auto-onboarding is enabled, new devices appear in the **Waiting Room** (not immediately as active environments):

1. Go to **Environments** → **Waiting Room**
2. Review new devices
3. Approve or reject each device
4. Approved devices become active environments

Or via API:

```bash
# List devices in waiting room
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/edge/waiting-room \
  | python3 -m json.tool

# Approve a device
DEVICE_ID="device-id-from-waiting-room"
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/edge/waiting-room/approve \
  -d "[\"$DEVICE_ID\"]"
```

## Fully Automated Onboarding (Skip Waiting Room)

For trusted environments, skip the waiting room:

1. Settings → Edge Compute
2. Enable **Skip waiting room for auto-onboarding**

Devices are immediately activated as environments when they connect.

## Conclusion

Automatic edge onboarding transforms device provisioning from a manual per-device process to an automated, scalable workflow. Pre-stage your edge key in device images, and new devices register themselves with Portainer on first boot. The Waiting Room provides a security checkpoint to review devices before granting full management access.
