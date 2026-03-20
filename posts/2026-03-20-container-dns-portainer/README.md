# How to Set Up Container DNS Resolution in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, DNS, Networking, Service Discovery, Container Communication

Description: Configure Docker's embedded DNS resolver for service discovery, custom DNS servers, and split-horizon DNS in containerized environments via Portainer.

## Introduction

Docker includes an embedded DNS server that runs at `127.0.0.11` inside every container. This server resolves container service names to IP addresses automatically. This guide covers Docker DNS internals, configuring custom DNS servers, split-horizon DNS, and troubleshooting DNS issues in containers managed by Portainer.

## How Docker DNS Works

```bash
Container A wants to reach "database"
    │
    ├─ Container checks /etc/resolv.conf
    │  → nameserver 127.0.0.11
    │
    ├─ Docker embedded DNS (127.0.0.11)
    │  → Resolves "database" to container IP
    │  → Falls back to host DNS for external names
    │
    └─ Returns IP → Container connects
```

## Step 1: View DNS Configuration in a Container

```bash
# View DNS settings inside a container

docker exec my_container cat /etc/resolv.conf

# Docker injects:
# nameserver 127.0.0.11
# options ndots:0

# Test DNS resolution
docker exec my_container nslookup database
docker exec my_container nslookup google.com
```

## Step 2: Configure Custom DNS Servers

```yaml
# docker-compose.yml - Custom DNS for containers
version: "3.8"

services:
  api:
    image: myapp/api:latest
    dns:
      # Primary: Use Pi-hole for ad blocking in containers
      - 192.168.1.250
      # Fallback: Cloudflare
      - 1.1.1.1
      # Secondary fallback: Google
      - 8.8.8.8
    dns_search:
      # Search domain for short hostnames
      - yourdomain.com
      - internal.yourdomain.com
    dns_opt:
      - ndots:1    # Reduce DNS lookup chains
      - timeout:5  # 5 second timeout
```

## Step 3: Global DNS Configuration for All Containers

```json
// /etc/docker/daemon.json - Docker daemon DNS settings
{
  "dns": ["192.168.1.250", "1.1.1.1"],
  "dns-search": ["yourdomain.com"],
  "dns-opts": ["ndots:1", "timeout:5", "attempts:3"]
}
```

```bash
# Apply changes
sudo systemctl restart docker

# Verify: new containers will use these DNS settings
docker run --rm alpine cat /etc/resolv.conf
```

## Step 4: Split-Horizon DNS for Internal Services

Configure containers to use internal DNS for private domains and external DNS for public domains:

```yaml
# docker-compose.yml - Split-horizon DNS
version: "3.8"

services:
  api:
    image: myapp/api:latest
    dns:
      # Internal DNS server handles *.internal.yourdomain.com
      - 10.0.0.1
      # External DNS for everything else
      - 1.1.1.1
    dns_search:
      # Auto-append for short names
      - internal.yourdomain.com
    environment:
      # Use full domain names in config
      - DB_HOST=postgres.internal.yourdomain.com
      - CACHE_HOST=redis.internal.yourdomain.com
```

Deploy an internal DNS server (Unbound) for split-horizon:

```yaml
  unbound:
    image: mvance/unbound:latest
    container_name: unbound_dns
    restart: unless-stopped
    ports:
      - "53:5335/udp"
      - "53:5335/tcp"
    volumes:
      - /opt/unbound:/opt/unbound/etc/unbound
```

```conf
# /opt/unbound/unbound.conf
server:
    interface: 0.0.0.0
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

# Forward internal domains to internal DNS
forward-zone:
    name: "internal.yourdomain.com."
    forward-addr: 10.0.0.53  # Your internal DNS server

# Forward everything else to Cloudflare
forward-zone:
    name: "."
    forward-addr: 1.1.1.1@853#cloudflare-dns.com
    forward-addr: 1.0.0.1@853#cloudflare-dns.com
    forward-tls-upstream: yes
```

## Step 5: Custom DNS Entries for Services

Add custom hostnames for services without a DNS server:

```yaml
# docker-compose.yml - Custom /etc/hosts entries
version: "3.8"

services:
  api:
    image: myapp/api:latest
    extra_hosts:
      # Map custom hostnames to IPs
      - "legacy-api.internal:192.168.1.100"
      - "payment-gateway.internal:192.168.1.101"
      - "s3-proxy.internal:192.168.1.102"
```

```bash
# Verify extra_hosts entries
docker exec api_container cat /etc/hosts
# Shows:
# 192.168.1.100  legacy-api.internal
# 192.168.1.101  payment-gateway.internal
```

## Step 6: DNS Caching for Performance

```yaml
# Reduce DNS lookup latency with local caching
  dnsmasq:
    image: jpillora/dnsmasq:latest
    container_name: dnsmasq
    restart: unless-stopped
    ports:
      - "5353:53/udp"
    volumes:
      - /opt/dnsmasq/dnsmasq.conf:/etc/dnsmasq.conf

# /opt/dnsmasq/dnsmasq.conf
# cache-size=1000
# no-resolv
# server=1.1.1.1
# server=8.8.8.8
# local=/internal.yourdomain.com/
# address=/db.internal.yourdomain.com/172.20.0.10
```

## Step 7: DNS Troubleshooting

```bash
# Debug DNS from inside a container
docker exec api_container bash -c "
    echo '=== /etc/resolv.conf ==='
    cat /etc/resolv.conf

    echo '=== DNS resolution ==='
    nslookup database
    nslookup google.com

    echo '=== DNS with specific server ==='
    nslookup database 127.0.0.11

    echo '=== Check iptables DNS rules ==='
    # This shows how Docker intercepts DNS
    iptables -t nat -L DOCKER_OUTPUT -n
"

# Watch DNS queries in real-time
docker run --rm --net=host nicolaka/netshoot tcpdump -n port 53

# Test from Portainer: Containers > Exec > nslookup service_name
```

## Conclusion

Docker's embedded DNS makes container service discovery automatic and reliable. For most use cases, the defaults work perfectly - service names resolve to container IPs within the same network. Custom DNS servers add ad blocking (Pi-hole), split-horizon DNS keeps internal names private, and extra_hosts handles edge cases where a DNS server isn't available. Portainer's container exec feature makes DNS debugging straightforward without SSH access.
