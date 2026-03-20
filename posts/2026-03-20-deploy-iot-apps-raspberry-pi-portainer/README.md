# How to Deploy IoT Applications on Raspberry Pi with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Raspberry Pi, IoT, Docker, ARM, Edge Computing

Description: Learn how to install Portainer on a Raspberry Pi and deploy IoT applications like Node-RED, MQTT, and Home Assistant in Docker containers.

---

The Raspberry Pi is a popular IoT gateway and edge computing device. Portainer simplifies deploying and managing Docker containers on Raspberry Pi, making it easy to run IoT software stacks without managing raw Docker commands.

---

## Install Docker on Raspberry Pi

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker pi
```

---

## Install Portainer on Raspberry Pi

```bash
# Create the volume

docker volume create portainer_data

# Deploy Portainer (ARM-compatible)
docker run -d   --name portainer   --restart=always   -p 9000:9000   -v /var/run/docker.sock:/var/run/docker.sock   -v portainer_data:/data   portainer/portainer-ce:latest
```

Access at `http://<pi-ip>:9000`.

---

## Deploy Node-RED for IoT Flows

In Portainer: **Stacks** → **Add stack**:

```yaml
version: "3.8"
services:
  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1880:1880"
    volumes:
      - nodered-data:/data
    environment:
      - TZ=America/New_York

volumes:
  nodered-data:
```

---

## Deploy Mosquitto MQTT Broker

```yaml
version: "3.8"
services:
  mosquitto:
    image: eclipse-mosquitto:latest
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - mosquitto-config:/mosquitto/config
      - mosquitto-data:/mosquitto/data
      - mosquitto-log:/mosquitto/log

volumes:
  mosquitto-config:
  mosquitto-data:
  mosquitto-log:
```

---

## Deploy Home Assistant

```yaml
version: "3.8"
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    privileged: true
    network_mode: host
    volumes:
      - ha-config:/config
      - /etc/localtime:/etc/localtime:ro

volumes:
  ha-config:
```

---

## Summary

Install Docker and Portainer on Raspberry Pi using the standard installation scripts (Portainer supports ARM). Deploy IoT stacks via Portainer's web UI with Node-RED for flow automation, Mosquitto as the MQTT broker, and Home Assistant for home automation. Use named volumes for persistent configuration storage and Portainer's GUI for monitoring container health on the Pi.
