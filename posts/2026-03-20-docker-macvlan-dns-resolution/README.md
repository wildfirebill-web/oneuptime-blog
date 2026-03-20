# How to Configure DNS Resolution in Docker macvlan Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Macvlan, DNS, Networking, IPv4, Containers

Description: Configure working DNS resolution for containers in Docker macvlan networks, which bypass Docker's built-in DNS server and require explicit DNS configuration.

## Introduction

Docker's `macvlan` driver assigns containers their own MAC and IP addresses on the physical network, making them appear as separate hosts. However, macvlan containers bypass Docker's embedded DNS server, so you must configure DNS explicitly - otherwise name resolution fails silently.

## The DNS Problem with macvlan

Docker's built-in DNS (`127.0.0.11`) is only available on user-defined bridge networks. macvlan containers skip this resolver entirely:

```bash
# On a macvlan container, Docker's DNS is not reachable

nslookup other-container
# Server: 127.0.0.11 -- this is NOT available in macvlan
```

## Creating a macvlan Network with DNS

Specify DNS servers when creating the network or at container launch:

```bash
# Create a macvlan network on the physical LAN
docker network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --ip-range 192.168.1.128/25 \
  -o parent=eth0 \
  macvlan-net
```

## Specifying DNS at Container Run Time

```bash
# Run a container on the macvlan network with explicit DNS servers
docker run -d \
  --network macvlan-net \
  --ip 192.168.1.130 \
  --dns 192.168.1.1 \           # Your LAN DNS or router
  --dns 8.8.8.8 \               # Public DNS fallback
  --dns-search example.local \  # Search domain for short names
  --name web nginx
```

## Using Docker Compose with macvlan and DNS

```yaml
# docker-compose.yml
version: "3.8"

services:
  web:
    image: nginx
    networks:
      macvlan-net:
        ipv4_address: 192.168.1.130
    dns:
      - 192.168.1.1      # Primary DNS (LAN resolver)
      - 8.8.8.8          # Fallback public DNS
    dns_search:
      - example.local

networks:
  macvlan-net:
    external: true       # Use the pre-created macvlan network
```

## Verifying DNS Resolution

```bash
# Enter the container and test DNS
docker exec -it web sh

# Test resolution inside the container
nslookup google.com 8.8.8.8
cat /etc/resolv.conf
```

## Using a Local DNS Resolver

For internal service discovery across macvlan containers, run a lightweight DNS resolver (e.g., dnsmasq) on the host and point containers to it:

```bash
# Run dnsmasq on the host with macvlan subnet visibility
# Point containers to the host's macvlan IP for DNS
docker run -d \
  --network macvlan-net \
  --ip 192.168.1.131 \
  --dns 192.168.1.1 \    # dnsmasq on the host at this IP
  --name app myapp
```

## Conclusion

macvlan networks provide excellent network isolation and performance but require explicit DNS configuration. Always specify `--dns` at run time or in Compose to ensure containers can resolve both internal and external names.
