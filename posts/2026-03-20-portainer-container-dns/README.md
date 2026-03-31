# How to Configure Container DNS Settings in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Container, Networking, DNS

Description: Learn how to configure custom DNS servers, search domains, and DNS options for Docker containers in Portainer.

## Introduction

By default, Docker containers inherit DNS settings from the host or use Docker's embedded DNS resolver. However, in enterprise environments, you may need containers to use specific DNS servers, custom search domains, or special DNS options. Portainer lets you configure all of these when creating a container.

## Prerequisites

- Portainer installed with a connected Docker environment
- Knowledge of your network's DNS infrastructure

## How Docker DNS Works

Docker's DNS resolution for containers:

1. **Bridge network**: Docker provides an embedded DNS resolver at `127.0.0.11` for service discovery (container name resolution).
2. **Host network**: Containers use the host's `/etc/resolv.conf` directly.
3. **Custom DNS**: You can override with specific DNS servers.

## Step 1: Configure DNS in Portainer

1. Navigate to **Containers > Add container**.
2. Scroll to the **Network** tab.
3. Look for the **DNS** section.
4. Configure:
   - **DNS server**: Custom DNS server IPs
   - **DNS search**: Search domain suffixes
   - **DNS options**: Additional resolver options

## Step 2: Set Custom DNS Servers

```yaml
# docker-compose.yml equivalent

services:
  app:
    image: myorg/myapp:latest
    dns:
      - 8.8.8.8          # Google DNS (primary)
      - 8.8.4.4          # Google DNS (secondary)

# Enterprise internal DNS:
  enterprise-app:
    image: myorg/app:latest
    dns:
      - 10.0.0.53        # Internal DNS server (primary)
      - 10.0.0.54        # Internal DNS server (secondary)
      - 8.8.8.8          # Public fallback
```

In Portainer:
```text
DNS server: 8.8.8.8
DNS server: 8.8.4.4
```

Click **+ add DNS server** for each server.

## Step 3: Set DNS Search Domains

Search domains allow you to use short hostnames instead of FQDNs:

```yaml
services:
  app:
    image: myapp:latest
    dns_search:
      - internal.example.com
      - svc.cluster.local
      - corp.mycompany.com
```

With `dns_search: internal.example.com`, the container can resolve `database` as `database.internal.example.com`.

In Portainer:
```text
DNS search: internal.example.com
DNS search: corp.mycompany.com
```

## Step 4: Set DNS Options

DNS options configure resolver behavior:

```yaml
services:
  app:
    image: myapp:latest
    dns_opt:
      - ndots:5          # Number of dots for absolute lookup
      - timeout:2        # Query timeout in seconds
      - attempts:3       # Retry attempts
      - rotate           # Rotate DNS servers (load balance)
```

In Portainer, add under **DNS options**:
```text
ndots: 5
timeout: 2
attempts: 3
```

## Step 5: Common DNS Configuration Scenarios

### Enterprise with Internal DNS

For containers that need to resolve internal services:

```yaml
services:
  internal-app:
    image: myorg/app:latest
    dns:
      - 10.0.0.10         # Primary internal DNS
      - 10.0.0.11         # Secondary internal DNS
    dns_search:
      - internal.mycompany.com
      - ad.mycompany.com
```

### Kubernetes-Style DNS (for containers that connect to K8s services)

```yaml
services:
  app:
    dns:
      - 10.96.0.10       # Kubernetes CoreDNS
    dns_search:
      - default.svc.cluster.local
      - svc.cluster.local
      - cluster.local
    dns_opt:
      - ndots:5
      - attempts:2
      - timeout:2
```

### High-Performance DNS (multiple resolvers)

```yaml
services:
  high-perf-app:
    dns:
      - 1.1.1.1           # Cloudflare (fast)
      - 1.0.0.1           # Cloudflare secondary
    dns_opt:
      - rotate            # Distribute queries across servers
      - timeout:1         # Fast timeout
      - attempts:2
```

### Private Network (Air-Gapped)

```yaml
services:
  airgapped-app:
    dns:
      - 192.168.1.53     # Local DNS only
    # No public DNS servers - all resolution stays internal
```

## Step 6: Set Global DNS Defaults in Docker Daemon

Instead of per-container DNS, configure defaults in the Docker daemon:

```json
// /etc/docker/daemon.json
{
  "dns": ["10.0.0.10", "10.0.0.11", "8.8.8.8"],
  "dns-search": ["internal.example.com"],
  "dns-opts": ["ndots:2", "timeout:2"]
}
```

```bash
# Apply and restart Docker:
systemctl restart docker
```

All new containers inherit these DNS settings unless overridden.

## Step 7: Verify DNS Configuration Inside Container

```bash
# In the container console (via Portainer Exec):

# Check /etc/resolv.conf:
cat /etc/resolv.conf
# nameserver 10.0.0.10
# nameserver 10.0.0.11
# search internal.example.com corp.mycompany.com
# options ndots:5

# Test DNS resolution:
nslookup google.com
dig internal-service.internal.example.com

# Test with specific DNS server:
nslookup google.com 8.8.8.8
```

## Troubleshooting DNS Issues

```bash
# Check if DNS is resolving inside the container:
docker exec my-container nslookup example.com

# Common issues:
# 1. DNS server unreachable: check firewall rules on port 53
# 2. Search domain not working: verify dns_search configuration
# 3. Slow DNS: add a fallback DNS server (8.8.8.8)

# Check Docker's embedded DNS:
docker exec my-container cat /etc/resolv.conf
# Should show: nameserver 127.0.0.11 (embedded DNS) on bridge networks
```

## Conclusion

DNS configuration in Portainer allows you to customize how containers resolve hostnames - critical for enterprise environments with internal DNS servers, private search domains, or specific DNS requirements. By configuring DNS per-container or via Docker daemon defaults, you ensure containers can reach the services they need without relying on public DNS that may be blocked or unavailable.
