# How to Use Portainer for Automotive Edge Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Automotive, Edge, V2X, Connected Vehicles

Description: Deploy and manage containerized applications at automotive edge infrastructure including vehicle-to-everything (V2X) roadside units, charging stations, and dealership networks using Portainer.

## Introduction

The automotive industry is deploying edge computing infrastructure at roadside units (RSUs), EV charging stations, dealerships, and manufacturing plants. These deployments share characteristics with industrial IoT: distributed locations, constrained hardware, intermittent connectivity, and long operational lifetimes. Portainer's Edge agent provides centralized management for containerized automotive applications across this diverse edge landscape.

## Automotive Edge Deployment Scenarios

- V2X (Vehicle-to-Everything) Roadside Units with containerized DSRC/CV2X services
- EV charging station management software
- Dealership connectivity and telematics platforms
- Over-the-air (OTA) update distribution servers
- Traffic management and analytics at intersections
- Assembly line quality control vision systems

## Step 1: Prepare Edge Hardware

Automotive edge hardware often runs on embedded x86 or ARM systems:

```bash
# Typical hardware: Intel NUC, NVIDIA Jetson, or automotive-grade PCs

# Install Docker on Ubuntu 20.04 (common in automotive embedded)
curl -fsSL https://get.docker.com | sh

# Configure for embedded/restricted environments
cat > /etc/docker/daemon.json << 'EOF'
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "userland-proxy": false
}
EOF

# Enable automatic recovery
systemctl enable docker
systemctl start docker

# Configure system watchdog for container recovery
cat > /etc/systemd/system/docker-watchdog.service << 'EOF'
[Unit]
Description=Docker watchdog
After=docker.service

[Service]
Type=simple
ExecStart=/usr/bin/docker events --filter event=die --format "{{.Actor.Attributes.name}}" | while read name; do docker start $name; done
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl enable docker-watchdog
```

## Step 2: Register Automotive Edge Nodes

```bash
#!/bin/bash
# register-edge-node.sh
NODE_TYPE=$1        # rsu, charging-station, dealership
NODE_ID=$2          # Unique identifier
LOCATION=$3         # GPS coordinates or address

PORTAINER_EDGE_KEY="your-portainer-edge-key"

# Add location metadata via Docker labels
docker run -d \
  --name portainer-agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v agent_data:/data \
  -e EDGE=1 \
  -e EDGE_ID="${NODE_TYPE}-${NODE_ID}" \
  -e EDGE_KEY="$PORTAINER_EDGE_KEY" \
  -e EDGE_SERVER_HOST="https://portainer.automotive.com:9443" \
  -l "automotive.node.type=$NODE_TYPE" \
  -l "automotive.node.id=$NODE_ID" \
  -l "automotive.location=$LOCATION" \
  portainer/agent:latest

echo "Registered: $NODE_TYPE-$NODE_ID at $LOCATION"
```

## Step 3: V2X Roadside Unit Stack

```yaml
# v2x-rsu/docker-compose.yml
version: '3.8'
services:
  v2x-stack:
    image: automotive/v2x-stack:v3.2.1
    restart: always
    network_mode: host   # Direct access to V2X radio hardware
    privileged: true     # Hardware access for DSRC/CV2X interface
    environment:
      - RSU_ID=${NODE_ID}
      - GEOGRAPHIC_REGION=US
      - SECURITY_CREDENTIALS_PATH=/certs
      - MAP_DATA_URL=https://maps.transportation.gov/v2x
      - SPAT_CONTROLLER_IP=192.168.1.100
    volumes:
      - /dev/dsrc:/dev/dsrc   # V2X radio device
      - rsu-certs:/certs
      - rsu-data:/data
    labels:
      - "automotive.service=v2x"

  traffic-monitor:
    image: automotive/traffic-analytics:v1.9
    restart: always
    environment:
      - RSU_ID=${NODE_ID}
      - CENTRAL_URL=https://traffic-ops.automotive.com
      - REPORTING_INTERVAL=60
      - VEHICLE_COUNT_ENABLED=true
    volumes:
      - traffic-data:/data

  cert-manager:
    image: automotive/scms-client:v2.0
    restart: always
    environment:
      - SCMS_URL=https://scms.automotive.com
      - CERT_LIFETIME_HOURS=168
      - AUTO_RENEW=true
    volumes:
      - rsu-certs:/certs

volumes:
  rsu-certs:
  rsu-data:
  traffic-data:
```

## Step 4: EV Charging Station Management

```yaml
# ev-charging/docker-compose.yml
version: '3.8'
services:
  ocpp-server:
    image: automotive/ocpp-server:v2.1
    restart: always
    ports:
      - "8080:8080"   # OCPP WebSocket
    environment:
      - STATION_ID=${NODE_ID}
      - CENTRAL_SYSTEM_URL=wss://csms.automotive.com/ocpp
      - MAX_CHARGING_POWER_KW=150
      - CONNECTOR_COUNT=4
    volumes:
      - ocpp-data:/data

  energy-monitor:
    image: automotive/energy-monitor:v1.4
    restart: always
    environment:
      - SMART_GRID_URL=https://grid.utility.com/api
      - PEAK_SHAVING_ENABLED=true
      - V2G_ENABLED=true   # Vehicle-to-Grid
      - TARIFF_API_URL=https://pricing.automotive.com

  local-display:
    image: automotive/charging-ui:v2.0
    restart: always
    ports:
      - "80:80"   # Local touchscreen UI

  payment-terminal:
    image: automotive/payment-proxy:v1.8
    restart: always
    environment:
      - PAYMENT_GATEWAY_URL=https://payments.automotive.com
      - RFID_READER_PORT=/dev/ttyUSB0
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0

volumes:
  ocpp-data:
```

## Step 5: OTA Update Distribution

```yaml
# ota-distribution/docker-compose.yml
version: '3.8'
services:
  ota-server:
    image: automotive/ota-server:v4.1
    restart: always
    ports:
      - "8443:8443"
    environment:
      - CENTRAL_REPO_URL=https://ota.automotive.com
      - SYNC_INTERVAL=3600   # Sync new packages hourly
      - MAX_CACHE_SIZE_GB=50
      - VEHICLE_PROTOCOLS=UDS,DoIP,SOME/IP
    volumes:
      - ota-packages:/data/packages
      - ota-cache:/data/cache
    deploy:
      resources:
        limits:
          memory: 2g

  vehicle-auth:
    image: automotive/vehicle-auth:v2.3
    restart: always
    environment:
      - PKI_URL=https://pki.automotive.com
      - VIN_VALIDATION=enabled
    volumes:
      - auth-data:/data
```

## Step 6: Fleet-Wide Deployment Management

```bash
#!/bin/bash
# deploy-to-fleet.sh - Update across all nodes by type
NODE_TYPE=$1   # rsu, charging-station, dealership
NEW_VERSION=$2
PORTAINER_URL="https://portainer.automotive.com"
API_KEY="fleet-ops-api-key"

echo "Deploying $NEW_VERSION to all $NODE_TYPE nodes..."

# Get edge group for this node type
EDGE_GROUP_ID=$(curl -s \
  -H "X-API-Key: $API_KEY" \
  "$PORTAINER_URL/api/edge_groups" | \
  python3 -c "
import sys, json
groups = json.load(sys.stdin)
for g in groups:
    if g['Name'] == '$NODE_TYPE':
        print(g['Id'])
        break
")

# Deploy update to edge group
curl -s -X PUT \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"StackFileContent\": \"$(cat ${NODE_TYPE}/docker-compose.yml | python3 -c 'import sys; import json; content = sys.stdin.read().replace(\"v[0-9.]*\", \"$NEW_VERSION\"); print(json.dumps(content))')\",
    \"EdgeGroups\": [$EDGE_GROUP_ID]
  }" \
  "$PORTAINER_URL/api/edge_stacks/1"

echo "Deployment initiated for $NODE_TYPE fleet"
```

## Step 7: Edge Node Health Monitoring

```bash
#!/bin/bash
# fleet-health.sh - Monitor automotive edge fleet health
PORTAINER_URL="https://portainer.automotive.com"
API_KEY="monitoring-api-key"

echo "=== Automotive Edge Fleet Health ==="
echo "$(date)"

# Get all edge environments grouped by type
curl -s \
  -H "X-API-Key: $API_KEY" \
  "$PORTAINER_URL/api/endpoints?types=4" | \
  python3 -c "
import sys, json
envs = json.load(sys.stdin)
types = {}
for e in envs:
    name = e['Name']
    node_type = name.split('-')[0] if '-' in name else 'unknown'
    if node_type not in types:
        types[node_type] = {'online': 0, 'offline': 0}
    if e.get('Status') == 1:
        types[node_type]['online'] += 1
    else:
        types[node_type]['offline'] += 1

for t, counts in sorted(types.items()):
    total = counts['online'] + counts['offline']
    pct = (counts['online'] / total * 100) if total > 0 else 0
    print(f'{t}: {counts[\"online\"]}/{total} online ({pct:.1f}%)')
"
```

## Conclusion

Automotive edge deployments share many characteristics with other industrial IoT deployments but add unique requirements: V2X protocol stacks, OCPP charging management, vehicle authentication PKI, and OTA update distribution. Portainer's Edge agent enables centralized management of these diverse containerized services across roadside units, charging stations, and dealerships from a single operational platform. Fleet-wide updates through Edge Stacks ensure consistent software versions across thousands of edge nodes, while health monitoring scripts provide the visibility needed to maintain automotive service availability.
