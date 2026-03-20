# How to Use Portainer in Manufacturing OT Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Manufacturing, OT, Industrial IoT, IIoT

Description: Deploy containerized applications on manufacturing shop floors alongside PLCs, CNC machines, and industrial robots using Portainer Edge agents for centralized OT management.

## Introduction

Modern manufacturing is embracing containerized applications for quality control, production monitoring, machine learning-based defect detection, and MES (Manufacturing Execution Systems) integration. However, manufacturing OT environments have unique constraints: ruggedized hardware, long equipment lifecycles, strict safety requirements, and network isolation. Portainer's Edge agent provides centralized management of containers deployed directly on the shop floor.

## Manufacturing Container Use Cases

- Quality inspection systems using computer vision
- Real-time production monitoring and OEE calculation
- Predictive maintenance using vibration/temperature sensors
- Digital twin applications for production simulation
- MES integration middleware
- Barcode/RFID tracking systems

## Step 1: Prepare Edge Hardware for Manufacturing

```bash
# Most manufacturing edge hardware runs on ARM or x86

# Example: Industrial PC at CNC machine

# Install Docker on Ubuntu 22.04 (ruggedized x86 PC)
curl -fsSL https://get.docker.com | sh
sudo systemctl enable docker
sudo systemctl start docker

# Configure Docker for reliability (manufacturing uptime is critical)
cat > /etc/docker/daemon.json << 'EOF'
{
  "live-restore": true,      
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "20m",
    "max-file": "5"
  },
  "icc": false,
  "storage-driver": "overlay2"
}
EOF

# Configure watchdog for automatic recovery
systemctl enable docker
systemctl enable --now docker
```

## Step 2: Deploy Portainer Edge Agent at Machine Level

```bash
# On the shop floor edge PC
MACHINE_ID="cnc-machine-042"
LINE_ID="assembly-line-b"
PLANT_ID="plant-chicago"

docker run -d \
  --name portainer-agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_agent_data:/data \
  -e EDGE=1 \
  -e EDGE_ID="${PLANT_ID}-${LINE_ID}-${MACHINE_ID}" \
  -e EDGE_KEY="your-edge-key-from-portainer" \
  -e EDGE_SERVER_HOST="https://portainer.manufacturing.com:9443" \
  portainer/agent:latest

echo "Edge agent registered: ${PLANT_ID}-${LINE_ID}-${MACHINE_ID}"
```

## Step 3: Quality Inspection System

```yaml
# quality-inspection/docker-compose.yml
version: '3.8'
services:
  vision-engine:
    image: mfg/quality-inspector:v2.4.1
    restart: always
    runtime: nvidia  # GPU acceleration for inference
    environment:
      - MODEL_VERSION=defect-classifier-v8
      - CAMERA_SOURCE=/dev/video0
      - DEFECT_THRESHOLD=0.92
      - MES_ENDPOINT=http://mes-server.plant.local/api/quality
      - LINE_ID=${LINE_ID}
      - MACHINE_ID=${MACHINE_ID}
    devices:
      - /dev/video0:/dev/video0   # Industrial camera
    volumes:
      - ./models:/models:ro
      - quality-logs:/var/log/quality
      - rejected-parts:/data/rejected  # Save images of defective parts
    networks:
      - plant-network

  oee-monitor:
    image: mfg/oee-calculator:v1.6
    restart: always
    environment:
      - PLC_IP=192.168.1.100
      - PLC_RACK=0
      - PLC_SLOT=1
      - SHIFT_START_HOUR=6
      - TARGET_PARTS_PER_HOUR=120
    networks:
      - plant-network

  data-bridge:
    image: mfg/mes-bridge:v3.1
    restart: always
    environment:
      - MES_URL=http://mes-server.plant.local
      - BUFFER_SIZE=1000
      - RETRY_INTERVAL=30
    volumes:
      - bridge-buffer:/var/buffer
    networks:
      - plant-network

volumes:
  quality-logs:
  rejected-parts:
  bridge-buffer:

networks:
  plant-network:
    driver: bridge
    internal: true
```

## Step 4: PLC Integration via Modbus/OPC-UA

```yaml
# plc-integration/docker-compose.yml
version: '3.8'
services:
  modbus-collector:
    image: iotech/device-modbus:2.1.0
    restart: always
    environment:
      - MODBUS_HOST=192.168.1.10
      - MODBUS_PORT=502
      - COLLECTION_INTERVAL=1000  # 1 second
    volumes:
      - ./config/device-modbus.yaml:/res/configuration.yaml:ro
    networks:
      - plant-network

  influxdb:
    image: influxdb:2.7-alpine
    restart: always
    volumes:
      - metrics-data:/var/lib/influxdb2
    networks:
      - plant-network

  telegraf:
    image: telegraf:1.27-alpine
    restart: always
    volumes:
      - ./telegraf-plc.conf:/etc/telegraf/telegraf.conf:ro
    networks:
      - plant-network
```

## Step 5: Update Management for Production Lines

```bash
#!/bin/bash
# production-update.sh
# Updates across machines while respecting production schedule

PORTAINER_URL="https://portainer.manufacturing.com"
API_KEY="ops-api-key"
EDGE_GROUP_ID=5  # Assembly Line B group

# Check if production is running before updating
check_production_status() {
  local machine_id=$1
  # Query OEE monitor for running status
  STATUS=$(curl -sf "http://$machine_id.plant.local/api/status" 2>/dev/null | \
    python3 -c "import sys,json; print(json.load(sys.stdin).get('running', False))" 2>/dev/null)
  echo $STATUS
}

# Only update machines that are not actively producing
MACHINES_TO_UPDATE=()
while read -r machine_id; do
  if [ "$(check_production_status $machine_id)" = "False" ]; then
    MACHINES_TO_UPDATE+=("$machine_id")
    echo "Scheduling update for idle machine: $machine_id"
  else
    echo "Skipping active machine: $machine_id"
  fi
done < machine-list.txt

# Deploy update to idle machines
if [ ${#MACHINES_TO_UPDATE[@]} -gt 0 ]; then
  curl -s -X POST \
    -H "X-API-Key: $API_KEY" \
    -H "Content-Type: application/json" \
    -d "{
      \"StackFileContent\": \"$(cat quality-inspection/docker-compose.yml | python3 -c 'import sys; import json; print(json.dumps(sys.stdin.read()))')\",
      \"EdgeGroups\": [$EDGE_GROUP_ID]
    }" \
    "$PORTAINER_URL/api/edge_stacks"
  echo "Update deployed to ${#MACHINES_TO_UPDATE[@]} machines"
fi
```

## Step 6: Alarm and Safety Integration

```yaml
# safety-monitor/docker-compose.yml
version: '3.8'
services:
  safety-monitor:
    image: mfg/safety-monitor:v1.2
    restart: always
    # Critical: highest restart priority
    deploy:
      restart_policy:
        condition: any
        delay: 1s    # Restart immediately on failure
        max_attempts: 0  # Never stop retrying
    environment:
      - SAFETY_PLC_IP=192.168.1.200
      - EMERGENCY_STOP_REGISTER=40001
      - ALARM_WEBHOOK=http://alarm-server.plant.local/api/alarm
    networks:
      - plant-network
```

## Step 7: OT/IT Data Integration

```bash
# Historian for OT-IT data bridge
# Runs in DMZ between OT and IT networks

cat > data-bridge/docker-compose.yml << 'EOF'
version: '3.8'
services:
  historian-bridge:
    image: mfg/historian-bridge:v4.0
    restart: always
    environment:
      - OT_INFLUX_URL=http://ot-influxdb.plant.local:8086
      - IT_INFLUX_URL=http://it-influxdb.corp.local:8086
      - REPLICATION_INTERVAL=60
      - FILTERED_METRICS=oee,defect_rate,parts_count,downtime
    networks:
      - ot-dmz
      - it-dmz

networks:
  ot-dmz:
    external: true
  it-dmz:
    external: true
EOF
```

## Conclusion

Manufacturing OT environments benefit from containerization for quality inspection, production monitoring, and predictive maintenance applications. Portainer's Edge agent enables centralized management of containers deployed at individual machine level across an entire factory floor. The production-aware update scripts ensure containers are only updated during machine idle time, preserving production continuity. Combined with robust restart policies and local data buffering, containerized manufacturing applications can meet the reliability standards demanded by industrial production environments.
