# How to Fix Swarm MTU Issues with Portainer on Hetzner

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Networking, MTU, Hetzner, DevOps

Description: Learn how to diagnose and fix MTU-related networking issues in Docker Swarm deployments on Hetzner Cloud infrastructure.

## Introduction

Hetzner Cloud uses a network infrastructure where the default MTU for overlay networks can cause packet fragmentation, resulting in intermittent connectivity issues between Swarm services. Symptoms include services that can ping each other but fail on larger payloads, or services that work fine initially but fail under load. This guide explains the root cause and how to fix it.

## The Problem

Hetzner Cloud's network interface has an MTU of 1450 bytes. Docker's default overlay network MTU is 1500 bytes. This mismatch means:

1. Packets larger than 1450 bytes get fragmented at the Hetzner network level
2. Many protocols and applications don't handle fragmentation well
3. This causes intermittent failures for larger HTTP requests, database queries, or API calls

## Symptoms

- Services can reach each other intermittently
- Small requests succeed, large requests fail
- Kubernetes pods or Swarm services timeout randomly
- `ping` works but `curl` fails with large responses
- gRPC connections drop unexpectedly

## Diagnosis

```bash
# Check current MTU on the network interface
ip link show eth0
# Look for: mtu 1450

# Check overlay network MTU
docker network inspect ingress --format '{{.Options}}'

# Test with PMTU discovery
ping -M do -s 1472 <remote-host>   # 1472 + 28 byte IP/ICMP header = 1500
ping -M do -s 1422 <remote-host>   # 1422 + 28 = 1450 (Hetzner limit)

# If the 1472 ping fails and 1422 succeeds, you have an MTU issue
```

## Step 1: Configure Docker Daemon MTU

Set the MTU in the Docker daemon configuration on **ALL** Swarm nodes:

```bash
# On each node, edit or create /etc/docker/daemon.json
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "mtu": 1400
}
EOF

# Restart Docker daemon
sudo systemctl restart docker

# Verify
docker info | grep -i mtu
```

Setting to 1400 gives headroom below Hetzner's 1450 limit (50 bytes for overhead).

## Step 2: Recreate the Overlay Networks

Existing overlay networks keep their old MTU. You must recreate them:

```bash
# List all overlay networks
docker network ls --filter driver=overlay

# Remove services using the networks first, then remove and recreate
# WARNING: This is disruptive; schedule maintenance

# Remove the default ingress network
docker network rm ingress

# Recreate with correct MTU
docker network create \
  --driver overlay \
  --ingress \
  --opt com.docker.network.driver.mtu=1400 \
  ingress

# Recreate custom overlay networks
docker network rm my-overlay-net
docker network create \
  --driver overlay \
  --opt com.docker.network.driver.mtu=1400 \
  my-overlay-net
```

## Step 3: Configure MTU in Compose/Stack Files

For Portainer stacks, specify MTU in the networks section:

```yaml
version: "3.8"

services:
  web:
    image: nginx:alpine
    networks:
      - app-net

  api:
    image: myapi:latest
    networks:
      - app-net

networks:
  app-net:
    driver: overlay
    driver_opts:
      com.docker.network.driver.mtu: "1400"    # Set MTU for this network
```

## Step 4: Configure MTU for New Swarm Stacks in Portainer

When deploying stacks via Portainer, include the MTU setting in your Compose file networks section. Portainer passes these options to the Docker Engine when creating networks.

1. Open your stack in Portainer
2. Edit the Compose file to add MTU driver options to each overlay network
3. Click **Update the stack**

## Step 5: Configure Docker Bridge Network MTU

Also fix the bridge network for standalone containers:

```json
// /etc/docker/daemon.json
{
  "mtu": 1400,
  "bip": "172.17.0.1/16"
}
```

## Step 6: Verify the Fix

```bash
# Check new MTU on overlay network
docker network inspect ingress --format '{{.Options}}'
# Should show: map[com.docker.network.driver.mtu:1400]

# Test connectivity with large packets between services
# From inside a container:
ping -M do -s 1372 other-service    # 1372 + 28 = 1400
```

## Step 7: Automate MTU Configuration with Cloud Init

For new Hetzner servers that will join the Swarm, configure MTU automatically:

```yaml
# cloud-init.yml for new Hetzner VMs
#cloud-config
write_files:
  - path: /etc/docker/daemon.json
    content: |
      {
        "mtu": 1400,
        "log-driver": "json-file",
        "log-opts": {
          "max-size": "10m",
          "max-file": "3"
        }
      }

runcmd:
  - systemctl restart docker
```

## Other Cloud Providers with MTU Issues

Similar issues occur on other providers:

| Provider | Network MTU | Recommended Docker MTU |
|----------|------------|----------------------|
| Hetzner Cloud | 1450 | 1400 |
| DigitalOcean VPC | 1500 | 1400 |
| AWS VPC | 9001 (jumbo) | 1500 |
| Vultr | 1500 | 1400 |
| Linode | 1500 | 1400 |

For AWS, the default 1500 usually works unless using VPN tunnels.

## Alternative: Use a Private Network

Hetzner supports private networks with configurable MTU. Create a private network with MTU 1450 and connect your VMs to it:

```bash
# In the Hetzner Cloud Console:
# 1. Create a Network with custom MTU
# 2. Attach all Swarm nodes to the private network
# 3. Use private IPs for Swarm communication
docker swarm init --advertise-addr <private-ip>
docker swarm join --token ... <manager-private-ip>:2377
```

## Conclusion

MTU mismatches are a common but easily overlooked cause of intermittent networking issues in Docker Swarm on Hetzner. The fix requires setting the Docker daemon MTU to a safe value (1400) on all nodes and recreating overlay networks with the correct MTU. Once configured, your Swarm services will have reliable networking for all payload sizes.
