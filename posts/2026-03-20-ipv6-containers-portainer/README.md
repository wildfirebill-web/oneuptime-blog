# How to Set Up IPv6 for Containers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, IPv6, Networking, Container Configuration

Description: Enable IPv6 networking for Docker containers, configure dual-stack networks, and test IPv6 connectivity in containers managed through Portainer.

## Introduction

IPv6 is disabled in Docker by default. With IPv6 adoption increasing and ISPs providing native IPv6 connectivity, containers may need IPv6 for direct internet access, compliance requirements, or internal IPv6 infrastructure. This guide covers enabling IPv6 in Docker, creating dual-stack networks, and deploying containers with IPv6 addresses via Portainer.

## Step 1: Enable IPv6 in Docker Daemon

```json
// /etc/docker/daemon.json - Enable IPv6 globally
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00::/80",
  "ip6tables": true,
  "experimental": false
}
```

```bash
# Apply daemon changes
sudo systemctl restart docker

# Verify IPv6 is enabled
docker network inspect bridge | grep -A 5 "EnableIPv6"
# Should show: "EnableIPv6": true
```

## Step 2: Create a Dual-Stack Network

```bash
# Create network with both IPv4 and IPv6 subnets
docker network create \
  --driver bridge \
  --subnet=172.30.0.0/24 \
  --gateway=172.30.0.1 \
  --ipv6 \
  --subnet=fd00::/80 \
  --gateway=fd00::1 \
  dual_stack_net

# Verify both address families are configured
docker network inspect dual_stack_net
```

## Step 3: Deploy Containers with IPv6 via Portainer

```yaml
# docker-compose.yml - Dual-stack deployment
version: "3.8"

networks:
  dual_stack:
    driver: bridge
    enable_ipv6: true
    ipam:
      driver: default
      config:
        # IPv4 range
        - subnet: 172.30.0.0/24
          gateway: 172.30.0.1
        # IPv6 range (ULA prefix - private)
        - subnet: fd00::/80
          gateway: fd00::1

services:
  nginx:
    image: nginx:alpine
    container_name: nginx_v6
    restart: unless-stopped
    networks:
      dual_stack:
        ipv4_address: 172.30.0.10
        ipv6_address: fd00::10   # Static IPv6 address
    ports:
      - "80:80"
      - "443:443"

  api:
    image: myapp/api:latest
    container_name: api_v6
    restart: unless-stopped
    networks:
      dual_stack:
        ipv4_address: 172.30.0.20
        ipv6_address: fd00::20   # Static IPv6 address
    environment:
      # Connect via IPv6
      - DB_HOST=fd00::30
      # Or use DNS (works for both IPv4 and IPv6)
      - DB_HOST=database

  database:
    image: postgres:15-alpine
    container_name: db_v6
    restart: unless-stopped
    networks:
      dual_stack:
        ipv4_address: 172.30.0.30
        ipv6_address: fd00::30   # Database accessible via IPv6
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=secure_pass
```

## Step 4: Test IPv6 Connectivity

```bash
# Check container's IPv6 address
docker exec nginx_v6 ip -6 addr show eth0
# Should show: fd00::10/80

# Test IPv6 between containers
docker exec api_v6 ping6 -c 3 fd00::30
docker exec api_v6 ping6 -c 3 database  # DNS resolves to IPv6 too

# Test IPv6 internet connectivity (requires host to have IPv6)
docker exec nginx_v6 ping6 -c 3 ipv6.google.com

# Curl with IPv6
docker exec nginx_v6 curl -6 http://[fd00::20]:8080/health

# Verify both IPv4 and IPv6 DNS resolution
docker exec api_v6 nslookup -query=AAAA database
docker exec api_v6 nslookup -query=A database
```

## Step 5: Expose Containers via IPv6

```yaml
# To listen on IPv6 as well as IPv4
services:
  nginx:
    image: nginx:alpine
    ports:
      # IPv4 binding
      - "0.0.0.0:80:80"
      # IPv6 binding
      - "[::]:80:80"
      # Or bind to all interfaces (both IPv4 and IPv6)
      - "80:80"  # Docker handles both by default when IPv6 enabled
```

```bash
# Verify nginx is listening on IPv6
docker exec nginx_v6 ss -tlnp | grep :80
# Should show :::80 (IPv6) and 0.0.0.0:80 (IPv4)

# Test from host
curl http://[::1]:80  # Loopback IPv6
curl http://[fd00::10]:80  # Container IPv6 address
```

## Step 6: IPv6 with NDP Proxy (Public IPv6)

For containers to get public IPv6 addresses from your ISP block:

```bash
# Example: Your ISP gives you 2001:db8::/48
# Allocate a /64 for containers

docker network create \
  --driver bridge \
  --ipv6 \
  --subnet=2001:db8:0:1::/64 \
  public_ipv6_net

# Enable NDP proxy so router knows about container IPs
sysctl -w net.ipv6.conf.eth0.proxy_ndp=1

# Add proxy entry for each container's IPv6
ip -6 neigh add proxy 2001:db8:0:1::10 dev eth0

# Now container is reachable from internet at 2001:db8:0:1::10
```

## Conclusion

IPv6 in Docker provides containers with globally routable addresses, eliminates NAT for internal services, and prepares your infrastructure for the IPv6-only future. Use ULA prefixes (fd00::/8) for private networks and public prefixes from your ISP for internet-facing services. Portainer's container inspection views show both IPv4 and IPv6 addresses, making it straightforward to verify dual-stack connectivity across your environment.
