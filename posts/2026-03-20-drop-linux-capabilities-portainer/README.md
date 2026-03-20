# How to Drop Unnecessary Linux Capabilities in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Security, Linux Capabilities, Container Hardening, Least Privilege

Description: Apply the principle of least privilege to Docker containers by dropping Linux capabilities that services don't need, reducing the attack surface in Portainer-managed deployments.

## Introduction

Linux capabilities break root privileges into distinct units. Docker grants containers a set of capabilities by default (including `CHOWN`, `NET_BIND_SERVICE`, `SETUID`, etc.) but not the full root privilege set. Dropping capabilities your application doesn't need further limits what a compromised container can do. A web server doesn't need `SYS_ADMIN`, a read-only API doesn't need `NET_BIND_SERVICE`, and most apps don't need `CHOWN`. This guide covers identifying and dropping unnecessary capabilities in Portainer.

## Step 1: Understand Docker's Default Capabilities

```bash
# Docker grants these capabilities by default:

# CHOWN          - Change file ownership
# DAC_OVERRIDE   - Bypass file permission checks
# FSETID         - Set file setuid/setgid bits
# FOWNER         - Bypass permission checks for file owner
# MKNOD          - Create special files
# NET_RAW        - Use raw sockets (can be abused for sniffing)
# SETGID         - Manipulate group IDs
# SETUID         - Manipulate user IDs
# SETFCAP        - Set file capabilities
# SETPCAP        - Transfer capabilities
# NET_BIND_SERVICE - Bind to ports < 1024
# SYS_CHROOT     - Use chroot
# KILL           - Send signals to processes
# AUDIT_WRITE    - Write to kernel audit log

# View capabilities of a running container
docker inspect my_container --format '{{.HostConfig.CapAdd}} {{.HostConfig.CapDrop}}'
```

## Step 2: Drop All Capabilities, Add Only What's Needed

The safest approach - start with nothing, add only what your app needs:

```yaml
# docker-compose.yml - Minimal capabilities
version: "3.8"

services:
  # Nginx on port 8080 (non-privileged) - needs almost nothing
  nginx:
    image: nginx:alpine
    cap_drop:
      - ALL  # Drop every capability
    cap_add:
      - NET_BIND_SERVICE  # Only if binding to port < 1024
      - CHOWN             # If nginx needs to chown files
      - SETUID            # To drop from root to nginx user
      - SETGID
    ports:
      - "8080:8080"

  # API server - read/write data, no special OS operations needed
  api:
    image: myapp/api:latest
    cap_drop:
      - ALL  # Drop everything
    # No cap_add needed - most apps work fine without any capabilities
    # when running as non-root user on ports > 1024

  # Database - needs more capabilities for memory management
  postgres:
    image: postgres:15-alpine
    cap_drop:
      - ALL
    cap_add:
      - CHOWN       # Set ownership on data files
      - FOWNER      # Access files regardless of owner
      - SETUID      # Drop from root to postgres user
      - SETGID
      - DAC_OVERRIDE  # Required for PostgreSQL initialization
```

## Step 3: Common Capability Requirements by Service Type

```yaml
version: "3.8"

services:
  # Reverse proxy (Traefik/Nginx on port 443)
  traefik:
    image: traefik:v3.0
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # Bind to 80/443

  # Redis - no special capabilities needed
  redis:
    image: redis:7-alpine
    cap_drop:
      - ALL
    # Redis on port 6379 as non-root needs zero capabilities

  # Pi-hole DNS server
  pihole:
    image: pihole/pihole:latest
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # DNS on port 53
      - NET_ADMIN         # iptables manipulation
      - SYS_NICE          # Priority management

  # WireGuard VPN (needs significant capabilities)
  wireguard:
    image: ghcr.io/wg-easy/wg-easy:latest
    cap_drop:
      - ALL
    cap_add:
      - NET_ADMIN         # Create TUN devices
      - SYS_MODULE        # Load WireGuard kernel module
```

## Step 4: Check If Your App Works Without Capabilities

```bash
# Test with all capabilities dropped
docker run --rm \
  --cap-drop ALL \
  --user 1000:1000 \
  myapp/api:latest \
  node server.js

# If it crashes, check what capability is missing
docker run --rm \
  --cap-drop ALL \
  --user 1000:1000 \
  myapp/api:latest \
  node -e "process.on('uncaughtException', e => console.error(e)); require('./server')"

# Use strace to identify permission denials
docker run --rm \
  --cap-drop ALL \
  --cap-add SYS_PTRACE \
  myapp/api:latest \
  strace -e trace=process node server.js 2>&1 | grep EPERM
```

## Step 5: Remove NET_RAW to Prevent Network Sniffing

`NET_RAW` is granted by default but allows raw socket access - a significant risk:

```yaml
services:
  api:
    image: myapp/api:latest
    cap_drop:
      - NET_RAW      # Drop just this dangerous capability
      - MKNOD        # Don't need to create device files
      - SYS_CHROOT   # Don't need chroot
      - AUDIT_WRITE  # Don't need kernel audit writes
      - SETFCAP      # Don't need to set file capabilities
```

```bash
# Verify NET_RAW is dropped
docker exec api_container ping google.com
# Error: ping: permission denied (uses raw ICMP)
# This confirms NET_RAW was successfully dropped
```

## Step 6: Audit Running Containers for Capability Excess

```bash
# List all running containers and their added capabilities
docker ps -q | while read id; do
  name=$(docker inspect "$id" --format '{{.Name}}')
  caps=$(docker inspect "$id" --format '{{.HostConfig.CapAdd}}')
  drop=$(docker inspect "$id" --format '{{.HostConfig.CapDrop}}')
  echo "$name: add=$caps drop=$drop"
done

# Flag containers with ALL capabilities (no cap_drop ALL)
docker ps -q | while read id; do
  drop=$(docker inspect "$id" --format '{{.HostConfig.CapDrop}}')
  name=$(docker inspect "$id" --format '{{.Name}}')
  if [[ "$drop" != *"ALL"* ]]; then
    echo "WARNING: $name has not dropped ALL capabilities"
  fi
done
```

## Conclusion

Dropping Linux capabilities is one of the most impactful container security hardening steps. The default Docker capability set is already restricted compared to full root, but `cap_drop: [ALL]` followed by adding only what's needed achieves true least privilege. Most modern web applications and databases run perfectly with no capabilities when configured to use non-root users and non-privileged ports. Portainer's stack editor makes it straightforward to update `cap_drop` and `cap_add` settings across all your services.
