# How to Drop Unnecessary Linux Capabilities in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Linux Capabilities, Container Security, Docker Hardening, Security

Description: Learn how to drop Linux capabilities from containers in Portainer to follow the principle of least privilege and reduce the container attack surface.

---

Linux capabilities divide root privileges into fine-grained units. Docker containers start with a limited set of capabilities by default. Dropping even more capabilities reduces the damage a compromised container can cause.

## Default Container Capabilities

Docker grants these capabilities by default:

| Capability | Purpose |
|------------|---------|
| `CHOWN` | Change file ownership |
| `DAC_OVERRIDE` | Bypass file permission checks |
| `FSETID` | Set SUID/SGID bits |
| `FOWNER` | Override file permission checks |
| `MKNOD` | Create device files |
| `NET_RAW` | Use raw sockets (ping, packet capture) |
| `SETGID` | Manipulate group IDs |
| `SETUID` | Manipulate user IDs |
| `SETFCAP` | Set file capabilities |
| `SETPCAP` | Manage process capabilities |
| `NET_BIND_SERVICE` | Bind to ports below 1024 |
| `SYS_CHROOT` | Use chroot |
| `KILL` | Send signals to processes |
| `AUDIT_WRITE` | Write to audit log |

## Dropping All Capabilities and Adding Back Only What's Needed

The most secure approach: drop everything, then explicitly add back what's required.

```yaml
version: "3.8"

services:
  api:
    image: my-api:latest
    user: "1000:1000"         # Run as non-root
    cap_drop:
      - ALL                   # Drop all default capabilities
    cap_add:
      - NET_BIND_SERVICE      # Only if binding to port < 1024
    read_only: true
    tmpfs:
      - /tmp
```

## Capabilities Reference by Use Case

| Application Type | Safe to Drop | May Need to Keep |
|------------------|--------------|-----------------|
| Web API (port 3000+) | ALL | None needed |
| Web server (port 80) | ALL except `NET_BIND_SERVICE` | `NET_BIND_SERVICE` |
| Monitoring agent | ALL | `SYS_PTRACE` (for some agents) |
| Database | `NET_RAW`, `MKNOD`, `AUDIT_WRITE` | `CHOWN`, `DAC_OVERRIDE` |
| Network tool | `MKNOD`, `SYS_CHROOT` | `NET_RAW`, `NET_ADMIN` |

## Dropping Specific High-Risk Capabilities

If you don't want to drop all, at minimum drop these high-risk capabilities:

```yaml
services:
  api:
    cap_drop:
      - NET_RAW        # Prevents raw socket access (used in ARP poisoning, ICMP flood)
      - NET_ADMIN      # Prevents network configuration changes
      - SYS_ADMIN      # Prevents many administrative operations (very broad)
      - SYS_PTRACE     # Prevents debugging other processes
      - SYS_MODULE     # Prevents loading kernel modules
      - DAC_READ_SEARCH  # Prevents bypassing filesystem permissions
```

## Configuring via Portainer UI

For standalone containers:

1. Go to **Containers > Add container**.
2. Expand **Runtime & Resources > Capabilities**.
3. Use the capability toggle list to enable/disable each capability.
4. All capabilities are shown; green = allowed, grey = dropped.

## Verifying Dropped Capabilities

Check the effective capabilities of a running container:

```bash
# View capabilities from inside the container

docker exec -it $(docker ps -qf name=api) cat /proc/1/status | grep Cap

# Decode the hex value
capsh --decode=00000000a80425fb

# Or use getpcaps
docker exec -it $(docker ps -qf name=api) getpcaps 1
```

## Testing Capability Restrictions

Verify that dropped capabilities prevent the expected operations:

```bash
# Test that NET_RAW is dropped (ping should fail)
docker exec -it $(docker ps -qf name=api) ping -c 1 8.8.8.8
# Should return: Operation not permitted

# Test that SYS_ADMIN is dropped (cannot mount)
docker exec -it $(docker ps -qf name=api) mount -t tmpfs none /tmp/test
# Should return: Operation not permitted
```

## Combining with No New Privileges

Prevent privilege escalation via SUID/SGID binaries:

```yaml
services:
  api:
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true   # Prevents privilege escalation
    user: "1000:1000"
```
