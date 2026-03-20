# How to Fix MAC Address Collisions in Docker Compose via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, MAC Address, Docker Compose, Networking, Macvlan

Description: Learn how to fix MAC address collision errors in Docker Compose stacks deployed via Portainer, particularly relevant for macvlan networks and static MAC assignments.

---

MAC address collisions in Docker occur when two containers or network interfaces are assigned the same MAC address. This is most common with macvlan networks or when manually specifying MAC addresses in Compose files.

## When MAC Collisions Occur

- Manually setting `mac_address` in Compose without ensuring uniqueness
- Cloning a VM or container that has a Docker macvlan interface
- Docker's random MAC generation produces a collision (rare but possible)

## Diagnosing a Collision

```bash
# Check if a stack failed with MAC-related errors

docker logs portainer 2>&1 | grep -i "mac\|duplicate\|collision"

# Or check Docker events
docker events --filter type=network --since 5m

# Check MAC addresses currently in use on a network
docker network inspect bridge | python3 -c "
import json, sys
data = json.load(sys.stdin)
for net in data:
    for cid, info in net['Containers'].items():
        print(f'{info[\"Name\"]}: {info[\"MacAddress\"]}')"
```

## Fixing Manual MAC Address Conflicts in Compose

If your Compose file sets `mac_address` explicitly, generate a unique one:

```bash
# Generate a random MAC address with the locally-administered bit set
python3 -c "
import random
mac = [0x02,                    # Locally administered unicast
       random.randint(0x00, 0xff),
       random.randint(0x00, 0xff),
       random.randint(0x00, 0xff),
       random.randint(0x00, 0xff),
       random.randint(0x00, 0xff)]
print(':'.join(f'{b:02x}' for b in mac))
"
```

Update the Compose YAML in Portainer with the new MAC:

```yaml
services:
  myservice:
    image: myimage
    networks:
      macvlan_net:
        # Use a unique MAC for each container on macvlan networks
        mac_address: "02:42:ac:11:00:02"
```

## Fixing Macvlan IP/MAC Pool Exhaustion

For macvlan networks, define explicit IP and MAC ranges to prevent overlap:

```yaml
networks:
  macvlan_net:
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      config:
        - subnet: 192.168.1.0/24
          ip_range: 192.168.1.200/28    # Only assign from .200-.215
          gateway: 192.168.1.1
```

## Preventing Collisions When Cloning Hosts

When cloning a VM that runs Docker with macvlan interfaces, reset the Docker daemon state:

```bash
# Stop Docker on the cloned VM
sudo systemctl stop docker

# Remove the network state files
sudo rm -rf /var/lib/docker/network/files/*

# Restart Docker - it will regenerate unique identifiers
sudo systemctl start docker
```
