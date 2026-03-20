# How to Pre-Stage Edge Agents with Join Tokens

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge Agent, Join Token, Pre-Staging, IoT, Mass Deployment

Description: Pre-configure edge agents with join tokens during device manufacturing or provisioning for zero-touch onboarding when devices first connect to the network.

## Introduction

Pre-staging edge agents means embedding the Portainer Edge Key and configuration into a device's provisioning image before deployment. When the device powers on and connects to the network, it automatically registers with Portainer without any on-site configuration. This is essential for mass deployments to retail stores, branch offices, or IoT deployments.

## Understanding Edge Keys vs Join Tokens

- **Edge Key**: Contains the full connection configuration (Portainer URL, device ID, tunnel server address) encoded as a base64 string
- **Join Token**: A shorter identifier used to associate the edge agent with a specific environment group or profile

## Generating an Edge Key

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# The Edge Key is generated when you create an edge environment in Portainer

# It's displayed in the deployment command

# Via API - create edge environment and get its key
EDGE_ENV=$(curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints \
  -d '{
    "name": "Pre-Staged Device Template",
    "endpointCreationType": 4,
    "isEdgeDevice": true
  }')

echo "Environment ID: $(echo $EDGE_ENV | python3 -c 'import sys,json; print(json.load(sys.stdin)["Id"])')"
echo "Edge Key: $(echo $EDGE_ENV | python3 -c 'import sys,json; print(json.load(sys.stdin).get("EdgeKey",""))')"
```

## Device Provisioning Script

Create a provisioning script to embed in your device image:

```bash
#!/bin/bash
# /opt/device-provision.sh
# This script runs on first boot to register the device with Portainer

# Configuration baked into the image
PORTAINER_URL="https://portainer.example.com"
EDGE_KEY="xxxxxx-base64-encoded-edge-key-xxxxxx"

# Generate device-unique ID from hardware
SERIAL=$(cat /sys/class/dmi/id/product_serial 2>/dev/null || hostname)
MAC=$(ip link | grep ether | awk '{print $2}' | head -1 | tr -d ':')
DEVICE_ID="device-${MAC}"

echo "Provisioning device: $DEVICE_ID"

# Install Docker if not present
which docker || curl -fsSL https://get.docker.com | sh

# Deploy Portainer Edge Agent
docker run -d \
  --name portainer_edge_agent \
  --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /var/run/portainer:/var/run/portainer \
  -e EDGE=1 \
  -e EDGE_ID="${DEVICE_ID}" \
  -e EDGE_KEY="${EDGE_KEY}" \
  portainer/agent:latest

echo "Device $DEVICE_ID registered with Portainer"

# Create provisioning marker so this script doesn't run again
touch /opt/.portainer-provisioned
```

## Systemd Service for Auto-Start

```bash
# /etc/systemd/system/portainer-provision.service
cat > /etc/systemd/system/portainer-provision.service << 'EOF'
[Unit]
Description=Portainer Edge Agent Provisioning
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStartPre=/bin/bash -c 'test ! -f /opt/.portainer-provisioned'
ExecStart=/opt/device-provision.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl enable portainer-provision.service
```

## Using a Generic Edge Key for Multiple Devices

Instead of creating one environment per device in advance, use a shared edge key that enables auto-onboarding:

1. Create one "template" edge environment with auto-onboarding enabled
2. Use that environment's edge key for all devices
3. Each device registers with a unique `EDGE_ID`
4. Devices appear in the Waiting Room for approval
5. Approved devices become independent environments

```bash
# All devices use the same EDGE_KEY, but unique EDGE_ID
# Device 1:
docker run -e EDGE_KEY=shared-key -e EDGE_ID=device-aabbccdd portainer/agent:latest

# Device 2:
docker run -e EDGE_KEY=shared-key -e EDGE_ID=device-11223344 portainer/agent:latest
```

## Conclusion

Pre-staging edge agents with join tokens/keys enables zero-touch deployment at scale. Bake the edge key and provisioning script into your device image during manufacturing or initial setup, and devices self-register with Portainer on first power-on. Combined with auto-onboarding and API-based bulk approval, this approach scales to thousands of devices with minimal operational overhead.
