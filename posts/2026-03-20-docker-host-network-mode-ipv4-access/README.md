# How to Use Docker Host Network Mode for Direct IPv4 Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Networking, Host Network, IPv4, Performance, Container

Description: Configure Docker containers to use the host network mode, sharing the host's IPv4 stack directly for maximum performance and simplified port management, with security trade-offs explained.

## Introduction

Docker's `host` network mode removes the container's network namespace and shares the Docker host's network stack directly. The container binds ports on the host's IP addresses without NAT, eliminating network overhead and simplifying firewall rules.

## Running a Container in Host Network Mode

```bash
# Run nginx using the host network - it listens on the host's port 80 directly

docker run -d \
  --name nginx-host \
  --network host \
  nginx:alpine

# No -p port mapping needed - nginx is directly on host:80
curl http://localhost:80
```

## Verifying the Container Uses Host Networking

```bash
# The container should show the host's IP, not a container IP
docker exec nginx-host hostname -I
# Output: same as `hostname -I` on the host itself

# Port 80 is bound on the host interface
ss -tlnp | grep :80
# Shows nginx binding to 0.0.0.0:80
```

## Docker Compose with Host Network

```yaml
# docker-compose.yml
version: "3.8"

services:
  app:
    image: my-high-performance-app:latest
    network_mode: host
    # No ports: section needed or allowed in host mode
```

Note: `ports:` mappings are ignored in `host` network mode.

## Use Cases for Host Networking

| Use Case | Reason |
|---|---|
| High-performance network applications | No NAT overhead, direct socket access |
| Network monitoring tools (tcpdump, sniffers) | Access to all host interfaces |
| Services needing low-latency UDP | No port translation delay |
| Applications binding to a specific source IP | Container sees host's real IP |

## Security Trade-offs

Host network mode removes container network isolation:
- Any port the container opens is directly on the host
- Container can access all host network interfaces
- A compromised container has full host network access

Use host mode only for trusted, internal workloads where network performance is critical.

## Checking Available Host Interfaces from Container

```bash
# From within a host-networked container
docker exec nginx-host ip addr show
# Shows all host interfaces (eth0, lo, docker0, etc.)
```

## Linux vs. Docker Desktop (macOS/Windows)

Host network mode only works as expected on **Linux**. On Docker Desktop (macOS and Windows), Docker runs inside a VM, so `--network host` gives the container the VM's network stack, not the actual macOS or Windows host.

## Conclusion

`--network host` eliminates Docker NAT and connects the container directly to the host's IPv4 stack. Use it for high-performance, low-latency applications or network tools. Be aware of the security implications and restrict use to trusted, well-monitored workloads.
