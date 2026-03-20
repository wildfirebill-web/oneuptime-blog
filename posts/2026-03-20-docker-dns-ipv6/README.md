# How to Configure Docker DNS Resolution for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, DNS, AAAA Records, Resolution, Container Networking

Description: Configure DNS resolution for IPv6 in Docker containers, set IPv6-capable DNS servers, understand how Docker's embedded DNS resolves container names over IPv6, and troubleshoot DNS AAAA resolution failures.

## Introduction

Docker containers use an embedded DNS server (127.0.0.11) for service discovery within user-defined networks. This DNS server resolves container names to their IPv6 addresses when IPv6 is enabled on the network. For external DNS resolution, containers use the host's DNS resolver by default. Configuring IPv6-capable DNS servers ensures containers can resolve AAAA records for external hostnames.

## Configure IPv6 DNS Servers

```json
// /etc/docker/daemon.json
{
  "ipv6": true,
  "ip6tables": true,
  "fixed-cidr-v6": "fd00:docker::/80",
  "dns": [
    "8.8.8.8",
    "8.8.4.4",
    "2001:4860:4860::8888",
    "2001:4860:4860::8844"
  ]
}
```

```bash
sudo systemctl restart docker

# Verify DNS servers are applied
docker run --rm alpine cat /etc/resolv.conf
# nameserver 8.8.8.8
# nameserver 8.8.4.4
# nameserver 2001:4860:4860::8888
```

## Per-Container DNS Configuration

```bash
# Override DNS for a specific container
docker run --rm \
    --dns 2606:4700:4700::1111 \
    --dns 2606:4700:4700::1001 \
    alpine nslookup -type=AAAA google.com

# Use multiple DNS servers (fallback order)
docker run --rm \
    --dns 2001:4860:4860::8888 \
    --dns 8.8.8.8 \
    alpine dig AAAA example.com

# Set DNS search domains
docker run --rm \
    --dns 8.8.8.8 \
    --dns-search internal.example.com \
    --dns-search example.com \
    alpine cat /etc/resolv.conf
```

## Docker Embedded DNS and IPv6 Service Discovery

```bash
# Create IPv6-enabled network with two services
docker network create \
    --driver bridge \
    --ipv6 \
    --subnet 172.20.0.0/24 \
    --subnet fd00:dns::/64 \
    dnstest

# Start two containers
docker run -d --name server --network dnstest nginx
docker run -d --name client --network dnstest alpine sleep infinity

# Test DNS resolution (Docker embedded DNS = 127.0.0.11)
docker exec client cat /etc/resolv.conf
# nameserver 127.0.0.11  <-- Docker embedded DNS

# Resolve server name to IPv6
docker exec client nslookup server
# Should return IPv6 address (fd00:dns::X)

# Or with dig
docker exec client sh -c "apk add bind-tools -q && dig AAAA server"

# Cleanup
docker rm -f server client
docker network rm dnstest
```

## Docker Compose DNS Configuration

```yaml
# compose.yaml

networks:
  appnet:
    driver: bridge
    enable_ipv6: true
    ipam:
      config:
        - subnet: 172.21.0.0/24
        - subnet: fd00:compose:dns::/64

services:
  web:
    image: nginx:latest
    networks:
      - appnet
    dns:
      - 2001:4860:4860::8888
      - 8.8.8.8
    dns_search:
      - internal.example.com

  api:
    image: myapi:latest
    networks:
      - appnet
    # Default DNS: inherits daemon.json DNS config
```

## Troubleshoot DNS AAAA Resolution

```bash
# Test DNS resolution for AAAA records
docker run --rm alpine sh -c "
    apk add --no-cache bind-tools -q
    echo '=== Testing AAAA resolution ==='
    dig AAAA google.com +short
    echo '=== Testing DNS server IPv6 ==='
    dig AAAA google.com @2001:4860:4860::8888 +short
    echo '=== Testing reverse IPv6 ==='
    dig -x 2001:4860:4860::8888 +short
"

# Common issue: DNS server unreachable over IPv6
# Fix: Ensure host has IPv6 connectivity
ping6 -c 3 2001:4860:4860::8888

# If IPv6 DNS server unreachable, use IPv4 DNS as fallback
docker run --rm \
    --dns 8.8.8.8 \
    alpine dig AAAA google.com
```

## Conclusion

Docker containers use `127.0.0.11` as the embedded DNS resolver for container name-to-IPv6 resolution within user-defined networks. Configure external IPv6-capable DNS servers (`2001:4860:4860::8888` for Google) in `daemon.json` under `"dns"` for AAAA record resolution. Per-container DNS overrides with `--dns` allow using different resolvers for specific containers. Docker Compose supports `dns` and `dns_search` under each service. Ensure the host has IPv6 connectivity before adding IPv6 DNS server addresses to `daemon.json`.
