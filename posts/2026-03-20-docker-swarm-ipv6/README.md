# How to Configure Docker Swarm with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Swarm, IPv6, Overlay Network, Cluster, Multi-Host

Description: Configure Docker Swarm clusters with IPv6 support, create IPv6-enabled overlay networks for swarm services, and enable container communication over IPv6 across swarm nodes.

## Introduction

Docker Swarm supports IPv6 through overlay networks, which span multiple hosts. To enable IPv6 in Swarm, all participating nodes must have IPv6 enabled in their Docker daemon configuration. Overlay networks in Swarm use VXLAN encapsulation and can carry IPv6 traffic between containers on different nodes. Swarm services deployed on IPv6 overlay networks receive IPv6 addresses in addition to their IPv4 addresses.

## Configure Each Swarm Node for IPv6

```json
// /etc/docker/daemon.json — apply to ALL swarm nodes (manager and workers)
{
  "ipv6": true,
  "ip6tables": true,
  "fixed-cidr-v6": "fd00:swarm:node1::/80",
  "default-address-pools": [
    {"base": "172.20.0.0/16", "size": 24},
    {"base": "fd00:swarm::/48", "size": 64}
  ]
}
```

```bash
# Apply on each node and restart
sudo systemctl restart docker

# Initialize Swarm (manager node)
docker swarm init --advertise-addr 192.168.1.10

# Join workers (on each worker node)
docker swarm join --token <token> 192.168.1.10:2377
```

## Create IPv6 Overlay Network

```bash
# Create overlay network with IPv6 (run on manager node)
docker network create \
    --driver overlay \
    --ipv6 \
    --subnet 10.1.0.0/24 \
    --subnet fd00:swarm:overlay:1::/64 \
    --gateway 10.1.0.1 \
    --gateway fd00:swarm:overlay:1::1 \
    swarm-overlay-net

# Verify network
docker network inspect swarm-overlay-net | grep -A10 "IPAM"

# List overlay networks
docker network ls --filter driver=overlay
```

## Deploy Services on IPv6 Overlay Network

```bash
# Deploy web service with IPv6
docker service create \
    --name web \
    --network swarm-overlay-net \
    --replicas 3 \
    --publish published=80,target=80 \
    nginx:latest

# Check service tasks and their IPv6 addresses
docker service ps web
docker inspect $(docker ps -q --filter name=web) | \
    python3 -c "
import json, sys
containers = json.load(sys.stdin)
for c in containers:
    for netname, netdata in c['NetworkSettings']['Networks'].items():
        if 'swarm' in netname.lower():
            print(f'Container: {c[\"Name\"][:30]}')
            print(f'  IPv4: {netdata.get(\"IPAddress\")}')
            print(f'  IPv6: {netdata.get(\"GlobalIPv6Address\")}')
"
```

## Docker Stack with IPv6

```yaml
# stack.yaml

networks:
  webnet:
    driver: overlay
    enable_ipv6: true
    ipam:
      config:
        - subnet: 10.2.0.0/24
        - subnet: fd00:swarm:stack:1::/64

  appnet:
    driver: overlay
    enable_ipv6: true
    ipam:
      config:
        - subnet: 10.2.1.0/24
        - subnet: fd00:swarm:stack:2::/64

services:
  nginx:
    image: nginx:latest
    networks:
      - webnet
    deploy:
      replicas: 3
    ports:
      - "80:80"

  api:
    image: myapi:latest
    networks:
      - webnet
      - appnet
    deploy:
      replicas: 2

  db:
    image: postgres:15
    networks:
      - appnet
    deploy:
      replicas: 1
```

```bash
# Deploy the stack
docker stack deploy -c stack.yaml mystack

# Check services
docker stack services mystack

# View task IPv6 addresses
docker service ps mystack_nginx
```

## Verify IPv6 Connectivity Across Swarm Nodes

```bash
# Test IPv6 connectivity between containers on different nodes
# Get IPv6 address of a container
CONTAINER_IPV6=$(docker inspect $(docker ps -q -f name=web) \
    --format "{{range .NetworkSettings.Networks}}{{.GlobalIPv6Address}}{{end}}" \
    2>/dev/null | head -1)

echo "Container IPv6: $CONTAINER_IPV6"

# From another container on a different node
docker exec $(docker ps -q -f name=api) \
    ping6 -c 3 "$CONTAINER_IPV6"
```

## Conclusion

Docker Swarm supports IPv6 overlay networks by enabling IPv6 in `daemon.json` on all nodes and creating overlay networks with `--ipv6 --subnet <ipv6-cidr>`. Services deployed on IPv6 overlay networks receive IPv6 addresses alongside IPv4. Use Docker Stack YAML with `enable_ipv6: true` and IPv6 IPAM configuration for declarative multi-service deployments. Ensure all swarm nodes have consistent IPv6 daemon configuration with non-overlapping ULA ranges for the local bridge networks.
