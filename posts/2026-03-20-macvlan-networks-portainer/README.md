# How to Configure Macvlan Networks for Direct LAN Access in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Macvlan, Networking, LAN, Home Lab, IoT

Description: Configure Macvlan networks to give containers their own MAC and IP addresses on your local network for direct LAN access via Portainer.

## Introduction

Macvlan networks assign each container its own MAC address and IP address on your physical LAN. This makes containers appear as physical devices on your network — useful for IoT applications, legacy systems that require direct LAN access, or services that need to be reachable on specific IP addresses. This guide covers configuring Macvlan via Portainer.

## When to Use Macvlan

Use Macvlan when you need:
- Containers to appear as physical devices on your LAN
- Specific IP addresses for containers (not port-mapped)
- Services that require Layer 2 network access
- Integration with network-aware applications (Bonjour/mDNS)
- IoT devices or home automation systems

## Prerequisites

```bash
# Check if your network interface supports promiscuous mode
ip link show eth0

# Enable promiscuous mode (required for Macvlan)
sudo ip link set eth0 promisc on

# Make it persistent
echo 'SUBSYSTEM=="net", ACTION=="add", KERNEL=="eth0", RUN+="/sbin/ip link set eth0 promisc on"' | \
  sudo tee /etc/udev/rules.d/99-macvlan.rules
```

## Step 1: Create Macvlan Network

```bash
# Create Macvlan network via Docker CLI
# Replace with your actual network details:
# - subnet: your home network subnet
# - gateway: your router IP
# - ip-range: IP range for Docker to assign (must not overlap with DHCP)

docker network create \
  --driver macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.240/28 \
  --opt parent=eth0 \
  lan_macvlan
```

**IP Range Explanation:**
- Your LAN: `192.168.1.0/24` (typical home network)
- Your router: `192.168.1.1`
- DHCP range: `192.168.1.2 - 192.168.1.239` (configured on router)
- Docker Macvlan range: `192.168.1.240 - 192.168.1.254` (excluded from DHCP)

## Step 2: Create Macvlan in Portainer

In Portainer: **Networks** > **Add network**
- Name: `lan_macvlan`
- Driver: `macvlan`
- Parent network card: `eth0`
- Subnet: `192.168.1.0/24`
- Gateway: `192.168.1.1`
- IP Range: `192.168.1.240/28`

## Step 3: Deploy Containers on Macvlan

```yaml
# docker-compose.yml - Containers on LAN with direct IPs
version: "3.8"

networks:
  # Reference the pre-created Macvlan network
  lan_macvlan:
    external: true

services:
  # Pi-hole with its own LAN IP address
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    restart: unless-stopped
    networks:
      lan_macvlan:
        ipv4_address: 192.168.1.250   # Static IP on your LAN!
    environment:
      - WEBPASSWORD=your_admin_password
      - PIHOLE_DNS_=1.1.1.1;9.9.9.9
      - TZ=America/New_York
    volumes:
      - /opt/pihole/etc:/etc/pihole
      - /opt/pihole/dnsmasq:/etc/dnsmasq.d
    cap_add:
      - NET_ADMIN

  # Home Assistant with its own IP
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    privileged: true
    networks:
      lan_macvlan:
        ipv4_address: 192.168.1.249   # Dedicated IP for HA
    environment:
      - TZ=America/New_York
    volumes:
      - /opt/homeassistant/config:/config
```

## Step 4: Enable Host-to-Container Communication

With Macvlan, the Docker host CANNOT communicate with Macvlan containers by default (limitation of Macvlan). Use a Macvlan bridge to fix this:

```bash
# Create a Macvlan sub-interface on the host
ip link add macvlan0 link eth0 type macvlan mode bridge
ip addr add 192.168.1.241/32 dev macvlan0  # Host's IP in Macvlan range
ip link set macvlan0 up

# Add route so host can reach Macvlan containers
ip route add 192.168.1.240/28 dev macvlan0

# Make persistent (Ubuntu/Debian - add to /etc/networkd-dispatcher/routable.d/)
cat > /etc/networkd-dispatcher/routable.d/50-macvlan << 'EOF'
#!/bin/sh
ip link add macvlan0 link eth0 type macvlan mode bridge
ip addr add 192.168.1.241/32 dev macvlan0
ip link set macvlan0 up
ip route add 192.168.1.240/28 dev macvlan0
EOF
chmod +x /etc/networkd-dispatcher/routable.d/50-macvlan
```

## Step 5: Mixed Network (Macvlan + Internal)

For containers that need both LAN access and internal Docker communication:

```yaml
# docker-compose.yml - Container with both Macvlan and bridge network
version: "3.8"

networks:
  lan_macvlan:
    external: true

  internal:
    driver: bridge

services:
  # This container is reachable from LAN AND from internal containers
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    networks:
      lan_macvlan:
        ipv4_address: 192.168.1.249
      internal:   # Also on internal network for container communication
    volumes:
      - /opt/homeassistant/config:/config

  # Internal container that connects to HA via internal network
  appdaemon:
    image: acockburn/appdaemon:latest
    networks:
      - internal   # Only on internal network
    environment:
      - HA_URL=http://homeassistant:8123  # Uses Docker DNS
```

## Step 6: Verify Macvlan is Working

```bash
# Check container IP on LAN
docker inspect pihole | jq '.[].NetworkSettings.Networks.lan_macvlan.IPAddress'
# Should return: 192.168.1.250

# Test from another device on your LAN
ping 192.168.1.250
curl http://192.168.1.250/admin  # Pi-hole admin

# From another device, check DNS
nslookup google.com 192.168.1.250
```

## Conclusion

Macvlan networks give your Docker containers a real presence on your home or office LAN. Pi-hole becomes accessible on a dedicated IP that every device can use as its DNS server, Home Assistant gets its own IP for easy integration with IoT devices, and any service that needs direct LAN access can have it. Portainer makes creating and managing Macvlan networks straightforward, and you can monitor all Macvlan containers from the same dashboard as your regular bridge network containers.
