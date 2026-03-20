# How to Configure Sysctls for Containers in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Linux, Networking

Description: Learn how to configure Linux kernel parameters (sysctls) for Docker containers in Portainer to tune network, memory, and IPC settings.

## Introduction

Sysctls are Linux kernel parameters that can be tuned at runtime via the `/proc/sys` filesystem. Docker allows setting container-namespaced sysctls to tune behavior for specific workloads - particularly useful for high-performance networking, database tuning, and real-time applications. Portainer exposes this configuration through its web interface.

## Prerequisites

- Portainer installed with a connected Docker environment running Linux
- Understanding of which sysctls are safe to modify in containers

## What Sysctls Can Be Set?

Docker only allows setting sysctls that are namespaced per-container (safe to change without affecting the host):

**Safe (namespaced) sysctls:**
- `net.ipv4.*` - TCP/IP settings
- `net.ipv6.*` - IPv6 settings
- `net.core.somaxconn` - Socket connection queue
- `net.unix.*` - Unix socket settings
- `kernel.msgmax`, `kernel.msgmnb`, `kernel.msgmni` - IPC message queues
- `kernel.sem` - Semaphores
- `kernel.shmmax`, `kernel.shmall` - Shared memory

**Unsafe (require `--privileged`):**
- `fs.*` - Filesystem parameters
- `kernel.sysrq` - System request key
- Most non-namespaced sysctls

## Step 1: Configure Sysctls in Portainer

1. Navigate to **Containers > Add container**.
2. Scroll to the **Runtime & Resources** section.
3. Look for the **Sysctls** section.
4. Click **+ add sysctl** for each parameter.
5. Enter the **Name** and **Value**.

## Step 2: Common Sysctl Use Cases

### High-Performance Web Server (Nginx, HAProxy)

```yaml
# docker-compose.yml

services:
  nginx:
    image: nginx:alpine
    restart: unless-stopped
    sysctls:
      # Allow more simultaneous connections
      net.core.somaxconn: 65535
      # Enable TCP fast open (reduce connection setup time)
      net.ipv4.tcp_fastopen: 3
      # Allow reuse of TIME_WAIT sockets (useful for high connection rate)
      net.ipv4.tcp_tw_reuse: 1
      # Reduce time_wait timeout
      net.ipv4.tcp_fin_timeout: 15
```

### Database Server (PostgreSQL, MySQL)

```yaml
services:
  postgres:
    image: postgres:15-alpine
    sysctls:
      # Increase shared memory limits for PostgreSQL
      kernel.shmmax: 268435456    # 256 MB
      kernel.shmall: 65536
      # IPC semaphores for PostgreSQL connections
      kernel.sem: "250 32000 100 128"
```

### Message Queue / Event Streaming (Redis, Kafka)

```yaml
services:
  redis:
    image: redis:7-alpine
    sysctls:
      # Disable THP (Transparent Huge Pages) notification
      # Redis recommends disabling THP for performance
      vm.overcommit_memory: 1
      # Increase socket buffer sizes
      net.core.rmem_max: 134217728
      net.core.wmem_max: 134217728
```

### UDP-Intensive Applications (DNS, Game Servers)

```yaml
services:
  dns-server:
    image: coredns/coredns:latest
    sysctls:
      # Increase UDP receive buffer
      net.core.rmem_default: 16777216
      net.core.rmem_max: 134217728
      net.core.wmem_default: 16777216
      net.core.wmem_max: 134217728
```

### Real-Time / Low-Latency Applications

```yaml
services:
  realtime-app:
    image: myorg/realtime:latest
    sysctls:
      # Minimize TCP latency
      net.ipv4.tcp_low_latency: 1
      # Disable Nagle algorithm (reduce latency for small packets)
      net.ipv4.tcp_nodelay: 1
```

## Step 3: Enable Sysctls in Docker Daemon (Alternative)

For sysctls that need to be set on all containers system-wide, or for unsafe sysctls:

```json
// /etc/docker/daemon.json
{
  "default-sysctls": {
    "net.ipv4.ip_local_port_range": "1024 65535"
  }
}
```

For unsafe sysctls, you need `--privileged` or must enable them in the daemon:

```bash
# Unsafe sysctl - requires privileged container or daemon config
docker run --privileged \
  --sysctl net.ipv4.ip_forward=1 \
  myimage
```

## Step 4: Verify Sysctls Inside the Container

After container creation:

```bash
# In the container console (via Portainer Exec):
sysctl net.core.somaxconn
# net.core.somaxconn = 65535

# List all current sysctls:
sysctl -a 2>/dev/null | grep net.core

# Compare with host (run on host):
sysctl net.core.somaxconn
# net.core.somaxconn = 128 (default host value, unchanged)
```

The container sysctl values are independent of the host.

## Important Restrictions

Some sysctls fail with an error in non-privileged containers:

```text
Error response from daemon: invalid argument "vm.swappiness=10"
for sysctl: not valid for kernel version
```

This means the sysctl is not namespaced and cannot be set per-container without `--privileged`.

## Security Considerations

- **Only set sysctls you understand** - incorrect values can degrade performance or cause instability.
- **Avoid unsafe sysctls** unless absolutely necessary, as they affect the host kernel.
- **Test sysctl changes** in a development environment before applying to production.
- **Document sysctl rationale** in your docker-compose.yml comments.

## Conclusion

Sysctls in Portainer provide a way to tune Linux kernel parameters for individual containers without affecting the host system. This is particularly valuable for high-performance applications like web servers, databases, and message queues that benefit from fine-tuned network or IPC settings. By using namespaced sysctls, you achieve per-container kernel tuning safely within Docker's isolation boundaries.
