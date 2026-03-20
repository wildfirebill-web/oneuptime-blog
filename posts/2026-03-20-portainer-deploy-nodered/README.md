# How to Deploy Node-RED via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Node-RED, IoT, Automation, Self-Hosted, Flow Programming

Description: Deploy Node-RED via Portainer as a flow-based programming tool for wiring IoT devices, APIs, and services together with a visual browser-based editor.

## Introduction

Node-RED is a flow-based programming tool originally developed by IBM for wiring IoT devices, APIs, and online services. It uses a visual browser-based editor to create flows and has a large ecosystem of community-contributed nodes. It's particularly popular for home automation alongside Home Assistant.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    environment:
      - TZ=America/New_York
      # Disable editor for production (set to false to enable)
      # - NODE_RED_ENABLE_PROJECTS=true
    volumes:
      # Node-RED data directory (flows, settings, credentials)
      - nodered_data:/data
    ports:
      - "1880:1880"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:1880/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  nodered_data:
```

## Securing Node-RED

Enable authentication in `settings.js`:

```bash
# Access Node-RED container
docker exec -it nodered bash

# Generate a password hash
node -e "console.log(require('bcryptjs').hashSync('your_password', 8))"
```

Add to `/data/settings.js`:

```javascript
adminAuth: {
    type: "credentials",
    users: [{
        username: "admin",
        password: "$2b$08$your_generated_hash",
        permissions: "*"
    }]
},
```

## Installing Additional Nodes

Via the Node-RED UI:

1. Click the hamburger menu > **Manage palette**
2. Click **Install** tab
3. Search for and install nodes

Popular packages:
- `node-red-contrib-mqtt` — MQTT support
- `node-red-contrib-influxdb` — InfluxDB integration
- `node-red-contrib-home-assistant-websocket` — Home Assistant
- `node-red-node-postgresql` — PostgreSQL
- `node-red-dashboard` — Dashboard UI

Or via command line:

```bash
docker exec nodered npm install node-red-dashboard
docker restart nodered
```

## Example Flows

### MQTT to InfluxDB Pipeline

```json
[
  {
    "id": "mqtt-in",
    "type": "mqtt in",
    "topic": "sensors/#",
    "broker": "mosquitto",
    "name": "Sensor Data"
  },
  {
    "id": "json-parse",
    "type": "json",
    "name": "Parse JSON"
  },
  {
    "id": "influx-out",
    "type": "influxdb out",
    "database": "sensors",
    "name": "Store in InfluxDB"
  }
]
```

### HTTP Webhook to Slack

```
HTTP In → Function → HTTP Request (Slack webhook) → HTTP Response
```

Function node:

```javascript
msg.payload = {
    text: `Webhook received: ${JSON.stringify(msg.payload)}`
};
msg.headers = {
    "Content-Type": "application/json"
};
msg.url = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL";
msg.method = "POST";
return msg;
```

## Node-RED with Home Assistant

```yaml
version: "3.8"

services:
  nodered:
    image: nodered/node-red:latest
    volumes:
      - nodered_data:/data
    # Need to be in the same network as Home Assistant
    network_mode: host
    ports:
      - "1880:1880"
    restart: unless-stopped
```

Install `node-red-contrib-home-assistant-websocket` to connect to your HA instance.

## Conclusion

Node-RED deployed via Portainer is an excellent IoT and automation tool that complements Home Assistant and works well in any data pipeline context. Its visual flow editor makes complex integrations accessible, and the large library of community nodes covers most integration needs. For home lab users, it's an excellent glue layer connecting sensors, databases, and notification services.
