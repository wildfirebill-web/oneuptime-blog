# How to Fix MAC Address Collisions in Docker Compose via Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Docker Compose, Troubleshooting, Networking

Description: Resolve MAC address collision errors in Docker Compose stacks deployed via Portainer, which cause containers to fail with network conflicts on recreate operations.

## Introduction

MAC address collisions in Docker occur when multiple containers are assigned the same MAC address on the same network. This can happen when containers are recreated from a compose file that explicitly sets `mac_address` values, or when Docker's automatic MAC assignment generates a duplicate. Portainer stack recreations are a common trigger.

## Understanding MAC Address Collisions

Docker automatically assigns unique MAC addresses to containers. Problems arise when:
1. A compose file explicitly sets `mac_address` and the container is recreated
2. The same compose file is deployed multiple times (creating duplicates)
3. Network bridges have conflicting MAC assignments

## Step 1: Identify the Error

```bash
# Look for MAC address errors in Portainer logs

docker logs portainer 2>&1 | grep -i "mac\|address already\|ARP"

# Check Docker daemon logs
journalctl -u docker | grep -i "mac\|duplicate\|conflict" | tail -20

# Check container creation errors
docker events --filter event=die --since 1h | head -20
```

## Step 2: Find Containers with Explicit MAC Addresses

```bash
# Find compose files with explicit MAC addresses
grep -r "mac_address" /opt/stacks/ 2>/dev/null

# List all running containers with their MAC addresses
docker ps -q | xargs docker inspect \
  --format '{{.Name}}: {{range .NetworkSettings.Networks}}{{.MacAddress}} {{end}}' | sort

# Find duplicates
docker ps -q | xargs docker inspect \
  --format '{{range .NetworkSettings.Networks}}{{.MacAddress}}{{end}}' | sort | uniq -d
```

## Step 3: Fix - Remove Explicit MAC Addresses from Compose Files

The simplest fix is to let Docker assign MAC addresses automatically:

```yaml
# BEFORE (problematic - causes collision on recreation)
version: "3.8"
services:
  myapp:
    image: myapp:latest
    networks:
      mynet:
        mac_address: "02:42:ac:11:00:02"  # Remove this

# AFTER (correct - Docker auto-assigns unique MAC)
version: "3.8"
services:
  myapp:
    image: myapp:latest
    networks:
      - mynet
```

In Portainer, edit the stack and remove the `mac_address` entries, then redeploy.

## Step 4: Fix - Assign Unique MAC Addresses

If you need explicit MAC addresses (e.g., for DHCP reservations), ensure uniqueness:

```yaml
version: "3.8"
services:
  app1:
    image: myapp:latest
    networks:
      mynet:
        # Use Docker's reserved prefix 02:42 with unique last octets
        mac_address: "02:42:ac:11:00:10"

  app2:
    image: myapp:latest
    networks:
      mynet:
        mac_address: "02:42:ac:11:00:11"  # Different last octet

  app3:
    image: myapp:latest
    networks:
      mynet:
        mac_address: "02:42:ac:11:00:12"
```

## Step 5: Fix - Reset the Network

If MAC collisions have corrupted the network state:

```bash
# Stop all containers using the network
docker compose down

# Remove the problematic network
docker network rm stack-name_network-name

# Recreate by bringing the stack back up
docker compose up -d
```

## Step 6: Fix via Portainer Stack Redeploy

In Portainer:
1. Go to **Stacks** → select the affected stack
2. Click **Editor** and remove/fix `mac_address` entries
3. Click **Update the stack** with **Re-pull image** enabled
4. Wait for the stack to redeploy

## Step 7: Check for Host Network MAC Conflicts

```bash
# Check the host's network interfaces
ip link show

# Check the Docker bridge interface
ip link show docker0

# If docker0 has a conflicting MAC with a physical interface
# (rare but possible on some virtualized environments)

# Reset docker0 MAC
sudo ip link set docker0 address 02:42:00:00:00:01

# More permanent fix via daemon config
cat > /etc/docker/daemon.json << 'EOF'
{
  "fixed-cidr": "172.17.0.0/16",
  "bip": "172.17.0.1/16"
}
EOF

sudo systemctl restart docker
```

## Step 8: Handle Macvlan Networks

Macvlan networks, which expose containers directly on the host network, are especially prone to MAC conflicts:

```yaml
# When using macvlan, ensure all containers have unique MACs
networks:
  macvlan-net:
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      driver: default
      config:
        - subnet: "192.168.1.0/24"
          gateway: "192.168.1.1"

services:
  myapp:
    image: myapp:latest
    networks:
      macvlan-net:
        # For macvlan, MAC must be unique on the physical network
        mac_address: "02:42:c0:a8:01:10"
        ipv4_address: "192.168.1.100"
```

## Step 9: Monitor for Future Conflicts

```bash
# Script to check for MAC duplicates periodically
#!/bin/bash
# check-mac-duplicates.sh

MACS=$(docker ps -q | xargs docker inspect \
  --format '{{.Name}}: {{range .NetworkSettings.Networks}}{{.MacAddress}} {{end}}' 2>/dev/null)

DUPLICATE_MACS=$(echo "$MACS" | awk '{print $2}' | sort | uniq -d)

if [ -n "$DUPLICATE_MACS" ]; then
  echo "WARNING: Duplicate MAC addresses detected:"
  echo "$DUPLICATE_MACS"
  echo ""
  echo "Containers with these MACs:"
  echo "$MACS" | grep -E "$(echo $DUPLICATE_MACS | tr ' ' '|')"
else
  echo "OK: No duplicate MAC addresses"
fi
```

## Conclusion

MAC address collisions in Docker Compose stacks deployed via Portainer are almost always caused by explicitly set `mac_address` values in compose files that conflict when containers are recreated. The simplest fix is to remove explicit MAC addresses and let Docker auto-assign them. If MAC assignment is required for DHCP reservations or macvlan networking, ensure each container has a genuinely unique MAC address.
