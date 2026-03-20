# How to Fix Swarm MTU Issues with Portainer on Hetzner

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker Swarm, Portainer, MTU, Hetzner, Networking, Troubleshooting

Description: Learn how to diagnose and fix MTU mismatch issues that cause network failures in Docker Swarm deployments on Hetzner Cloud.

## The Problem: MTU Mismatches on Hetzner

Hetzner Cloud uses a network MTU of **1450 bytes** (due to its underlying network virtualization), while Docker's default overlay network MTU is **1500 bytes**. This mismatch causes packet fragmentation, dropped packets, and mysterious connectivity failures between Swarm services.

Symptoms include:
- Services timing out randomly
- Large payloads failing while small ones succeed
- DNS resolution working but HTTP requests hanging

## Diagnosing the Issue

```bash
# Check the MTU of your host network interface

ip link show eth0

# Check the MTU of Docker overlay networks
docker network inspect ingress | grep -i mtu

# Test connectivity with a specific packet size (if ping fails, MTU is the issue)
ping -M do -s 1400 <target-ip>
```

## Fix 1: Configure Docker Daemon MTU

Edit the Docker daemon configuration to set the default MTU globally:

```json
// /etc/docker/daemon.json
{
  "mtu": 1450
}
```

Apply the change:

```bash
# Restart Docker to apply the new MTU setting
sudo systemctl restart docker
```

## Fix 2: Set MTU Per Network in Compose File

You can set the MTU on individual overlay networks in your stack's Compose file:

```yaml
version: "3.8"

services:
  web:
    image: nginx:alpine
    networks:
      - hetzner-net

networks:
  hetzner-net:
    driver: overlay
    driver_opts:
      # Match Hetzner's network MTU to avoid packet fragmentation
      com.docker.network.driver.mtu: "1450"
```

## Fix 3: Re-Create the Ingress Network

If your Swarm was already initialized before the MTU was fixed, re-create the default ingress network:

```bash
# Remove the old ingress network (services using it must be removed first)
docker network rm ingress

# Re-create ingress with the correct MTU
docker network create \
  --driver overlay \
  --ingress \
  --opt com.docker.network.driver.mtu=1450 \
  ingress
```

## Verifying the Fix

```bash
# Confirm the overlay network now has the correct MTU
docker network inspect ingress | grep -i mtu

# Test large payload connectivity between containers
docker exec -it <container-id> ping -M do -s 1400 <other-container-ip>
```

## Conclusion

MTU mismatches are a common and frustrating issue on Hetzner Cloud. Always set `mtu: 1450` in your Docker daemon configuration and in your overlay network definitions to prevent connectivity problems in Swarm deployments.
