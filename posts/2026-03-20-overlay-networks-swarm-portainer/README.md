# How to Set Up Overlay Networks for Swarm Services in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker Swarm, Overlay Networks, Portainer, Networking, Container Networking

Description: Learn how to create and configure overlay networks for Docker Swarm services using Portainer.

## What Are Overlay Networks?

Overlay networks connect multiple Docker daemons together, enabling containers running on different Swarm nodes to communicate as if they were on the same local network. They are essential for multi-service Swarm deployments.

## Creating an Overlay Network in Portainer

1. Navigate to your Swarm environment in Portainer.
2. Go to **Networks** in the sidebar.
3. Click **Add network**.
4. Set the **Driver** to `overlay`.
5. Configure the subnet (optional) and click **Create the network**.

## Key Options When Creating an Overlay Network

| Option | Description |
|--------|-------------|
| **Attachable** | Allows standalone containers (not just Swarm services) to attach |
| **Encrypted** | Enables data plane encryption between nodes |
| **Subnet** | Custom CIDR, e.g., `10.0.9.0/24` |
| **Gateway** | Optional custom gateway IP |

## Creating an Overlay Network via CLI

```bash
# Create a basic overlay network
docker network create --driver overlay my-overlay-net

# Create an encrypted overlay network (data plane encryption)
docker network create \
  --driver overlay \
  --opt encrypted \
  my-secure-net

# Create an attachable overlay (allows standalone containers)
docker network create \
  --driver overlay \
  --attachable \
  my-attachable-net
```

## Attaching Services to an Overlay Network

In your stack Compose file, reference the overlay network under each service:

```yaml
version: "3.8"

services:
  frontend:
    image: nginx:alpine
    networks:
      - my-overlay-net  # Attach to the overlay network

  backend:
    image: my-api:latest
    networks:
      - my-overlay-net  # Both services share the same overlay

networks:
  my-overlay-net:
    external: true  # Reference an existing network created in Portainer
```

## Checking Network Connectivity

```bash
# List all networks in the Swarm
docker network ls

# Inspect a specific network to see connected services
docker network inspect my-overlay-net
```

## Troubleshooting Overlay Networks

- **Port 4789/UDP** must be open between Swarm nodes (VXLAN traffic).
- **Port 7946/TCP and UDP** must be open for control plane traffic.
- If services cannot communicate, verify they are on the same overlay network using `docker network inspect`.

## Conclusion

Overlay networks are the backbone of Swarm service communication. Using Portainer, you can create and manage these networks visually, making multi-node service configuration straightforward.
