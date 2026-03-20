# How to Expose Docker Container Ports on IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Port Binding, Expose, Published Ports

Description: Publish Docker container ports on IPv6 addresses, configure port bindings for both IPv4 and IPv6, and handle the IPv6 wildcard address for dual-stack port exposure.

## Introduction

Docker port publishing (`-p` / `--publish`) binds container ports to the host's network interfaces. By default, ports bind to both IPv4 (`0.0.0.0`) and IPv6 (`::`) when IPv6 is enabled. You can explicitly bind ports to specific IPv6 addresses using `[IPv6]:host_port:container_port` syntax. Understanding Docker's port binding behavior with IPv6 is important for controlling which interfaces accept connections.

## Basic IPv6 Port Publishing

```bash
# Publish port 80 on all interfaces (IPv4 + IPv6)
docker run -d \
    -p 80:80 \
    --name web \
    nginx:latest

# Verify port bindings
docker port web
# 80/tcp -> 0.0.0.0:80
# 80/tcp -> [::]:80  (IPv6 wildcard)

# Test IPv4 access
curl http://localhost/

# Test IPv6 access
curl -6 http://[::1]/
```

## Explicit IPv6 Port Binding

```bash
# Bind to IPv6 wildcard (all IPv6 interfaces)
docker run -d \
    -p "[::]:80:80" \
    --name web-ipv6only \
    nginx:latest

# Bind to specific IPv6 address only
docker run -d \
    -p "[2001:db8::1]:80:80" \
    --name web-specific \
    nginx:latest

# Bind to localhost IPv6 only (loopback)
docker run -d \
    -p "[::1]:8080:80" \
    --name web-local \
    nginx:latest

# Verify bindings
docker port web-ipv6only
docker port web-specific
docker port web-local
```

## Dual-Stack Port Publishing in Docker Compose

```yaml
# compose.yaml

services:
  web:
    image: nginx:latest
    ports:
      # Publish on all IPv4 and IPv6 interfaces
      - "80:80"
      - "443:443"

  api:
    image: myapi:latest
    ports:
      # Explicit IPv6 and IPv4 bindings
      - "0.0.0.0:8080:8080"  # IPv4 only
      - "[::]:8080:8080"      # IPv6 only

  admin:
    image: myadmin:latest
    ports:
      # Only bind on localhost IPv6 (local access only)
      - "[::1]:9090:9090"
```

## Check Actual Port Bindings

```bash
# List all Docker port bindings
docker ps --format "table {{.Names}}\t{{.Ports}}"

# Using ss to see actual IPv6 listening sockets
ss -tlnp6 | grep docker

# Check specific container port mapping
docker inspect web | python3 -c "
import json, sys
data = json.load(sys.stdin)
ports = data[0]['NetworkSettings']['Ports']
for port, bindings in ports.items():
    if bindings:
        for b in bindings:
            print(f'Container {port} -> Host {b[\"HostIp\"]}:{b[\"HostPort\"]}')
"
```

## IPv6 Port Binding with userland-proxy

```bash
# By default, Docker uses userland-proxy for port binding
# The proxy handles IPv6 -> container translation

# Check if userland-proxy is running
ps aux | grep docker-proxy

# To disable userland-proxy (uses iptables directly):
# daemon.json:
# { "userland-proxy": false }

# With userland-proxy disabled, Docker uses ip6tables NAT
# to redirect IPv6 ports directly

# Verify with netstat
ss -tlnp | grep :80
# Shows: [::]:80 or 0.0.0.0:80 depending on configuration
```

## Conclusion

Docker publishes container ports on IPv6 using `[::]:port:container_port` syntax. By default, `-p 80:80` binds to both `0.0.0.0:80` (IPv4) and `[::]:80` (IPv6) when IPv6 is enabled. Use explicit `[IPv6]:host_port:container_port` format to bind to specific IPv6 addresses or the IPv6 wildcard only. Verify port bindings with `docker port <container>` and check actual listening sockets with `ss -tlnp6`. The userland-proxy process handles the IPv6-to-container packet forwarding.
