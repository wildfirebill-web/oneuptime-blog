# How to Deploy Home Assistant via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Home Assistant, Home Automation, IoT, Self-Hosted

Description: Deploy Home Assistant via Portainer for a powerful, open-source home automation platform that integrates thousands of smart home devices and services.

## Introduction

Home Assistant is the leading open-source home automation platform, supporting 3000+ integrations with smart home devices and services. Deploying via Portainer with host network mode enables proper device discovery and local integration support.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    # Host network is required for:
    # - mDNS/Zeroconf device discovery
    # - Local device communication
    # - UPnP/DLNA
    network_mode: host
    environment:
      - TZ=America/New_York
    volumes:
      # Home Assistant configuration
      - homeassistant_config:/config
      # Required for time synchronization
      - /etc/localtime:/etc/localtime:ro
    # Uncomment if using USB devices (Zigbee/Z-Wave dongles)
    # devices:
    #   - /dev/ttyUSB0:/dev/ttyUSB0   # Zigbee/Z-Wave stick
    #   - /dev/ttyACM0:/dev/ttyACM0
    privileged: true   # Required for some USB device access
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:8123/api/ -H 'Authorization: Bearer TOKEN' | grep -q 'message'"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 60s

volumes:
  homeassistant_config:
```

## Initial Setup

1. Access Home Assistant at `http://<host>:8123`
2. Complete the onboarding wizard:
   - Create your user account
   - Set your location (for sunrise/sunset automation)
   - Install recommended integrations

## Adding Integrations

1. Navigate to **Settings > Devices & Services**
2. Click **Add integration**
3. Search for your device type (Philips Hue, Nest, MQTT, etc.)
4. Follow the integration-specific setup

## Example Automation

In **Settings > Automations > Create automation**:

```yaml
# automation.yaml - Turn on lights at sunset
alias: "Turn on lights at sunset"
description: ""
trigger:
  - platform: sun
    event: sunset
    offset: "-00:30:00"
condition:
  - condition: time
    weekday:
      - mon
      - tue
      - wed
      - thu
      - fri
action:
  - service: light.turn_on
    target:
      area_id: living_room
    data:
      brightness_pct: 70
      color_temp: 400
mode: single
```

## Zigbee Integration via USB Dongle

If using a Zigbee coordinator (ConBee II, HUSBZB-1, etc.):

```yaml
services:
  homeassistant:
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    privileged: true

  # ZHA alternative: Zigbee2MQTT
  zigbee2mqtt:
    image: koenkk/zigbee2mqtt:latest
    container_name: zigbee2mqtt
    volumes:
      - zigbee2mqtt_data:/app/data
      - /run/udev:/run/udev:ro
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    environment:
      TZ: America/New_York
    network_mode: host
    restart: unless-stopped
```

## MQTT Broker for Home Assistant

Many devices communicate via MQTT. Add Mosquitto to your stack:

```yaml
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto
    ports:
      - "1883:1883"
    volumes:
      - mosquitto_config:/mosquitto/config
      - mosquitto_data:/mosquitto/data
    restart: unless-stopped
```

## Backing Up Home Assistant

```bash
# Create a backup via Home Assistant CLI
docker exec homeassistant ha backups new --name "manual-backup"

# Copy backup from container
docker cp homeassistant:/config/backups ./ha-backups/
```

Or in the UI: **Settings > System > Backups > Create backup**

## Conclusion

Home Assistant deployed via Portainer creates a central hub for your smart home. The host network mode ensures device discovery works correctly for local protocols like mDNS, SSDP, and direct device communication. Portainer's stack management makes updating Home Assistant straightforward, and the persistent configuration volume preserves your automations and integrations across updates.
