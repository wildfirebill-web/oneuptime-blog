# How to Configure Podman with IPv6 Networking

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, IPv6, Container Networking, CNI, Rootless, Linux

Description: A guide to configuring Podman container networking with IPv6 support, including custom networks, dual-stack configuration, and rootless IPv6 containers.

Podman uses CNI (Container Network Interface) plugins for container networking, which supports IPv6 natively. Both root and rootless Podman can use IPv6 networks with appropriate configuration.

## Enabling IPv6 in Podman

Podman's default network configuration may not include IPv6. Enable it per-network:

```bash
# Check default network configuration
podman network inspect podman

# Check if IPv6 is enabled on default bridge
cat /etc/cni/net.d/87-podman.conflist | python3 -m json.tool | grep ipv6
```

## Creating an IPv6-Enabled Network

```bash
# Create a dual-stack network
podman network create \
  --driver bridge \
  --subnet 10.88.0.0/16 \
  --gateway 10.88.0.1 \
  --ipv6 \
  --subnet fd00:podman::/64 \
  --gateway fd00:podman::1 \
  ipv6-network

# IPv6-only network
podman network create \
  --ipv6 \
  --subnet fd00:podman-ipv6::/64 \
  --gateway fd00:podman-ipv6::1 \
  ipv6-only

# List networks
podman network ls

# Inspect IPv6 network
podman network inspect ipv6-network
```

## Running Containers with IPv6

```bash
# Run a container on the IPv6 network
podman run -d \
  --name web \
  --network ipv6-network \
  -p 80:80 \
  nginx:alpine

# Check container's IPv6 address
podman inspect web | python3 -m json.tool | grep -A 5 "IPAddress\|IPv6"

# Or using exec
podman exec web ip -6 addr show

# Test IPv6 connectivity to container
curl -6 http://[fd00:podman::2]/
```

## Pods with IPv6

```bash
# Create a pod with IPv6 networking
podman pod create \
  --name my-pod \
  --network ipv6-network \
  -p 8080:80

# Run containers in the pod
podman run -d \
  --pod my-pod \
  --name web \
  nginx:alpine

podman run -d \
  --pod my-pod \
  --name app \
  my-app:latest

# Check pod network configuration
podman pod inspect my-pod | python3 -m json.tool | grep -A 10 "Networks"
```

## Podman Compose with IPv6

```yaml
# docker-compose.yml (works with podman-compose)

version: '3.9'

networks:
  ipv6:
    enable_ipv6: true
    driver: bridge
    ipam:
      config:
        - subnet: 10.30.0.0/24
        - subnet: fd00:compose::/64

services:
  web:
    image: nginx:alpine
    networks:
      - ipv6
    ports:
      - "80:80"

  app:
    image: my-app:latest
    networks:
      - ipv6
```

```bash
# Deploy with podman-compose
podman-compose up -d

# Verify IPv6 addresses
podman-compose ps
```

## Rootless Podman with IPv6

Rootless Podman requires `slirp4netns` which has limited IPv6 support, or the newer `pasta` networking:

```bash
# Using pasta (better IPv6 support for rootless)
# Install pasta
sudo apt-get install passt

# Run rootless container with pasta networking
podman run --network pasta --name web nginx:alpine

# Or configure pasta as default in containers.conf
mkdir -p ~/.config/containers
cat >> ~/.config/containers/containers.conf << 'EOF'
[network]
default_rootless_network_cmd = "pasta"
EOF

# Verify rootless container has IPv6
podman exec web ip -6 addr show
```

## Configuring the Default Network for IPv6

```json
// /etc/cni/net.d/87-podman.conflist
{
  "cniVersion": "0.4.0",
  "name": "podman",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni-podman0",
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "10.88.0.0/16"}],
          [{"subnet": "fd00:podman::/64"}]
        ],
        "routes": [
          {"dst": "0.0.0.0/0"},
          {"dst": "::/0"}
        ]
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    }
  ]
}
```

## Verifying IPv6 Container Networking

```bash
# List all container IPv6 addresses
podman ps -q | xargs -I{} podman inspect {} --format '{{.Name}}: {{range .NetworkSettings.Networks}}{{.GlobalIPv6Address}}{{end}}'

# Test inter-container IPv6 communication
podman exec web ping6 -c 3 fd00:podman::app-address

# Check IPv6 routing from container
podman exec web ip -6 route show
```

Podman's CNI-based networking with explicit IPv6 subnet configuration provides reliable dual-stack container networking, with the newer pasta networking backend improving rootless IPv6 support.
