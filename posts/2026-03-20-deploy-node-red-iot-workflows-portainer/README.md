# How to Deploy Node-RED for IoT Workflows via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Node-RED, IoT, Portainer, Docker, Automation, MQTT, Workflow

Description: Deploy Node-RED on Docker using Portainer to create visual IoT data processing workflows that connect sensors, MQTT brokers, databases, and cloud services without writing code.

---

Node-RED is a flow-based visual programming tool that makes it easy to wire together IoT devices, APIs, and services. It runs on Node.js and ships as a Docker image, making it a perfect candidate for Portainer deployment.

## Step 1: Deploy Node-RED via Portainer Stack

Navigate to **Stacks > Add Stack** in Portainer:

```yaml
# node-red-stack.yml

version: "3.8"

services:
  node-red:
    image: nodered/node-red:3.1.3
    environment:
      # Set timezone for accurate timestamps in flows
      - TZ=America/New_York
    volumes:
      # Persistent storage for flows and node configurations
      - node-red-data:/data
    ports:
      - "1880:1880"    # Node-RED editor and HTTP endpoints
    restart: unless-stopped
    user: "1000"       # Run as non-root for security
    networks:
      - iot-net

networks:
  iot-net:
    driver: bridge

volumes:
  node-red-data:
```

## Step 2: Install Additional Nodes

After deploying, access the Node-RED editor at `http://<host>:1880`. Install extra nodes via the **Palette Manager** (Menu > Manage Palette > Install):

- `node-red-contrib-mqtt-broker` - embedded MQTT broker
- `node-red-contrib-influxdb` - write directly to InfluxDB
- `node-red-dashboard` - build web-based dashboards
- `node-red-contrib-postgresql` - PostgreSQL integration

Alternatively, pre-install nodes in the Docker image using a custom `package.json`:

```json
{
  "name": "my-node-red",
  "description": "Node-RED with pre-installed IoT nodes",
  "dependencies": {
    "node-red-contrib-influxdb": "0.6.1",
    "node-red-dashboard": "3.6.0",
    "node-red-contrib-postgresql": "0.14.0"
  }
}
```

Mount this file to `/data/package.json` and Node-RED will install the packages on startup.

## Step 3: Create an IoT Data Flow

Here is an example flow exported as JSON that reads MQTT sensor data and writes it to InfluxDB:

```json
[
  {
    "id": "mqtt-in",
    "type": "mqtt in",
    "name": "Sensor Input",
    "topic": "sensors/#",
    "qos": "1",
    "broker": "mqtt-broker-config",
    "wires": [["json-parse"]]
  },
  {
    "id": "json-parse",
    "type": "json",
    "name": "Parse JSON",
    "wires": [["influxdb-out"]]
  },
  {
    "id": "influxdb-out",
    "type": "influxdb out",
    "name": "Write to InfluxDB",
    "influxdb": "influxdb-config",
    "measurement": "sensor_readings",
    "wires": []
  }
]
```

Import this flow via **Menu > Import > Clipboard** in the Node-RED editor.

## Step 4: Secure Node-RED

By default, Node-RED has no authentication. Enable it by editing `settings.js`:

```javascript
// /data/settings.js in the container
module.exports = {
  // Enable HTTP authentication for the editor
  adminAuth: {
    type: "credentials",
    users: [
      {
        username: "admin",
        // Generate hash: node -e "console.log(require('bcryptjs').hashSync('your-password', 8))"
        password: "$2b$08$your-bcrypt-hash-here",
        permissions: "*"
      }
    ]
  },

  // Enable HTTPS (mount certs as volumes)
  https: {
    key: require("fs").readFileSync("/certs/server.key"),
    cert: require("fs").readFileSync("/certs/server.crt")
  }
};
```

## Step 5: Backup Flows

Node-RED flows are stored in `/data/flows.json` inside the container. With Portainer, you can:

1. Use the **Volume backup** feature to export `node-red-data`
2. Copy `/data/flows.json` out of the container using Portainer's file browser
3. Use git-backed flow storage with the `node-red-contrib-git` node

## Summary

Node-RED on Portainer is an excellent combination for IoT workflow automation. The visual flow editor lowers the barrier to building data pipelines, and Portainer makes it easy to keep the Node-RED instance running, updated, and backed up.
