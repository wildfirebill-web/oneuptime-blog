# How to Deploy Node-RED for IoT Workflows via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Node-RED, IoT, Automation, Docker

Description: Deploy Node-RED as a visual IoT workflow automation platform using Portainer for low-code device data processing and integration.

## Introduction

Node-RED is a flow-based programming tool built on Node.js that makes it easy to wire together IoT devices, APIs, and online services. It provides a browser-based editor where you can create data processing workflows by dragging and connecting nodes. Deploying Node-RED via Portainer gives IoT teams a powerful workflow automation platform that's easy to manage and scale.

## Prerequisites

- Portainer installed with Docker
- Basic understanding of Node.js and IoT protocols
- At least 512 MB RAM available for Node-RED

## Step 1: Deploy Node-RED via Portainer Stack

Create a new stack in Portainer:

```yaml
# docker-compose.yml for Node-RED
version: "3.8"

services:
  nodered:
    image: nodered/node-red:3.1.3-18
    container_name: nodered
    restart: always
    user: "1000"
    ports:
      - "1880:1880"
    volumes:
      # Persist Node-RED flows and settings
      - nodered-data:/data
    environment:
      - TZ=UTC
      - NODE_RED_ENABLE_PROJECTS=true
      - NODE_RED_ENABLE_SAFE_MODE=false
      # Credential encryption key
      - NODE_RED_CREDENTIAL_SECRET=${NODE_RED_CREDENTIAL_SECRET}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:1880/"]
      interval: 30s
      timeout: 10s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "3"
    networks:
      - iot-net

  # Optional: Include Mosquitto for local MQTT
  mosquitto:
    image: eclipse-mosquitto:2.0
    container_name: mosquitto
    restart: always
    ports:
      - "1883:1883"
    volumes:
      - mosquitto-data:/mosquitto/data
    networks:
      - iot-net

volumes:
  nodered-data:
  mosquitto-data:

networks:
  iot-net:
    driver: bridge
```

## Step 2: Configure Node-RED Settings

Create a settings file via Portainer's Configs:

```javascript
// settings.js - Node-RED configuration
module.exports = {
    // Web UI settings
    uiPort: process.env.PORT || 1880,
    
    // Flow file location
    flowFile: 'flows.json',
    
    // Credential encryption
    credentialSecret: process.env.NODE_RED_CREDENTIAL_SECRET,
    
    // Enable function node code coverage
    functionGlobalContext: {
        // Add global variables available in all function nodes
        moment: require('moment'),
    },
    
    // User authentication
    adminAuth: {
        type: "credentials",
        users: [{
            username: "admin",
            password: "$2b$08$your-bcrypt-hash",
            permissions: "*"
        }]
    },
    
    // Editor settings
    editorTheme: {
        projects: {
            enabled: true
        },
        palette: {
            allowInstall: true,
            editable: true
        }
    },
    
    // Context storage
    contextStorage: {
        default: {
            module: "localfilesystem"
        }
    },
    
    // Logging
    logging: {
        console: {
            level: "info",
            metrics: false,
            audit: false
        }
    }
};
```

## Step 3: Install IoT-Specific Node-RED Nodes

After deployment, install additional nodes via the Portainer container console:

```bash
# Access Node-RED container via Portainer console
# Navigate to Containers > nodered > Console

# Install MQTT nodes (usually pre-installed)
npm install --prefix /data node-red-node-mqtt

# Install InfluxDB nodes for time-series storage
npm install --prefix /data node-red-contrib-influxdb

# Install Modbus nodes for industrial protocols
npm install --prefix /data node-red-contrib-modbus

# Install OPC-UA nodes
npm install --prefix /data node-red-contrib-opcua

# Install dashboard nodes for UI
npm install --prefix /data node-red-dashboard

# Restart Node-RED to load new nodes
# (Portainer: Containers > nodered > Restart)
```

## Step 4: Example IoT Workflow

Here's a Node-RED flow that reads MQTT sensor data and stores it in InfluxDB:

```json
[
    {
        "id": "mqtt-input",
        "type": "mqtt in",
        "name": "Sensor MQTT Input",
        "topic": "sensors/#",
        "qos": "1",
        "broker": "mosquitto-broker",
        "x": 150,
        "y": 100
    },
    {
        "id": "parse-json",
        "type": "json",
        "name": "Parse JSON",
        "x": 350,
        "y": 100
    },
    {
        "id": "transform",
        "type": "function",
        "name": "Transform for InfluxDB",
        "func": "// Transform sensor payload to InfluxDB line protocol\nconst data = msg.payload;\nconst topic = msg.topic.split('/');\nconst deviceId = topic[1] || 'unknown';\n\n// Create InfluxDB measurement\nmsg.payload = [{\n    measurement: 'sensor_data',\n    tags: {\n        device_id: deviceId,\n        location: data.location || 'unknown'\n    },\n    fields: {\n        temperature: parseFloat(data.temperature),\n        humidity: parseFloat(data.humidity),\n        pressure: parseFloat(data.pressure || 0)\n    },\n    timestamp: data.timestamp ? new Date(data.timestamp * 1000) : new Date()\n}];\n\nreturn msg;",
        "x": 550,
        "y": 100
    },
    {
        "id": "influxdb-out",
        "type": "influxdb out",
        "name": "Write to InfluxDB",
        "influxdb": "influxdb-config",
        "measurement": "sensor_data",
        "x": 750,
        "y": 100
    }
]
```

Import this flow via Node-RED UI: Menu > Import > Clipboard.

## Step 5: Create an Alerting Flow

Add alerting when sensor values exceed thresholds:

```javascript
// Function node: Check sensor thresholds
const data = msg.payload;
const alerts = [];

// Temperature threshold check
if (data.temperature > 35) {
    alerts.push({
        severity: 'critical',
        message: `HIGH TEMPERATURE: ${data.temperature}°C on device ${data.device_id}`,
        timestamp: new Date().toISOString()
    });
} else if (data.temperature > 30) {
    alerts.push({
        severity: 'warning',
        message: `Elevated temperature: ${data.temperature}°C on device ${data.device_id}`,
        timestamp: new Date().toISOString()
    });
}

if (alerts.length > 0) {
    msg.payload = alerts[0];
    return msg;
}
return null; // No alert needed
```

## Step 6: Expose Node-RED Safely

For production access, use Nginx reverse proxy configured in Portainer:

```yaml
# Add to your stack
  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - nodered
```

```nginx
# nginx.conf
server {
    listen 443 ssl;
    server_name nodered.example.com;
    
    ssl_certificate /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;
    
    # Proxy to Node-RED
    location / {
        proxy_pass http://nodered:1880;
        proxy_http_version 1.1;
        # WebSocket support for Node-RED editor
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

## Conclusion

Node-RED deployed via Portainer provides a powerful visual programming environment for IoT data workflows. Its drag-and-drop interface makes it accessible to developers and non-developers alike, while its extensible node library supports virtually any IoT protocol and cloud service. Portainer simplifies the deployment, configuration management, and monitoring of Node-RED, making it easy to run production-grade IoT automation at scale.
