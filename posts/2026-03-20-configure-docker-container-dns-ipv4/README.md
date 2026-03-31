# How to Configure Docker Container DNS Settings for IPv4 Resolution

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Networking, DNS, IPv4, Container, Configuration

Description: Configure custom DNS servers, search domains, and DNS options for Docker containers using --dns flags and daemon.json to control how containers resolve IPv4 hostnames.

## Introduction

By default, Docker containers use the Docker daemon's DNS settings, which typically point to `8.8.8.8` or the host's resolver. Customizing DNS per container or daemon-wide is essential in corporate environments with internal DNS servers or split-horizon DNS.

## Setting DNS for a Single Container

```bash
# Run a container with custom DNS servers

docker run -d \
  --name my-app \
  --dns 192.168.1.10 \
  --dns 192.168.1.11 \
  --dns-search corp.example.com \
  --dns-opt ndots:5 \
  nginx:alpine

# Verify from inside the container
docker exec my-app cat /etc/resolv.conf
```

Expected `/etc/resolv.conf` in the container:

```text
search corp.example.com
nameserver 192.168.1.10
nameserver 192.168.1.11
options ndots:5
```

## Setting Default DNS in daemon.json

To apply DNS settings to all containers:

```bash
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "dns": ["192.168.1.10", "8.8.8.8"],
  "dns-search": ["corp.example.com", "internal"],
  "dns-opts": ["ndots:5", "timeout:2"]
}
EOF

sudo systemctl restart docker
```

## Docker Compose DNS Configuration

```yaml
# docker-compose.yml
version: "3.8"

services:
  web:
    image: nginx:alpine
    dns:
      - 192.168.1.10
      - 8.8.8.8
    dns_search:
      - corp.example.com
    dns_opt:
      - ndots:5
      - timeout:2
```

## Testing DNS Resolution Inside a Container

```bash
# Test resolution from inside a container
docker exec my-app nslookup internal-server.corp.example.com

# Or with dig
docker exec my-app dig +short internal-server.corp.example.com

# Test with a specific server
docker exec my-app nslookup internal-server.corp.example.com 192.168.1.10
```

## Docker's Embedded DNS

By default, Docker containers on user-defined networks use Docker's internal DNS at `127.0.0.11`. This resolves container names within the same network:

```bash
# Docker's embedded DNS server
docker exec my-app cat /etc/resolv.conf
# nameserver 127.0.0.11

# Resolves other containers by name
docker exec my-app ping db-service
```

When you set `--dns`, Docker adds those servers after `127.0.0.11`, so container name resolution still works.

## Troubleshooting DNS in Containers

```bash
# Check if the DNS server is reachable from the container
docker exec my-app nc -uvz 192.168.1.10 53

# Check iptables rules for DNS
sudo iptables -L DOCKER-USER -n -v | grep 53

# Check the container's resolv.conf
docker exec my-app cat /etc/resolv.conf
```

## Conclusion

Use `--dns` flags in `docker run`, the `dns:` key in Docker Compose, or `daemon.json` to configure DNS at different scopes. Docker's embedded DNS (`127.0.0.11`) handles container-to-container name resolution on user-defined networks and coexists with custom DNS servers.
