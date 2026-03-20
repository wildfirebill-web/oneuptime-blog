# How to Configure AppArmor Profiles for Containers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Security, AppArmor, Linux Security, Container Hardening

Description: Create and apply AppArmor profiles to restrict container capabilities at the kernel level, controlling file access, network operations, and capability usage via Portainer.

## Introduction

AppArmor is a Linux Security Module that enforces access control policies through profiles. Docker applies a default AppArmor profile (`docker-default`) to all containers on supported systems. Custom profiles let you restrict specific containers to only the filesystem paths, network operations, and capabilities they actually need. This guide covers creating and deploying custom AppArmor profiles for containers managed by Portainer.

## Prerequisites

```bash
# Check if AppArmor is enabled

sudo aa-status
# Should show: apparmor module is loaded

# Check if Docker is using AppArmor
docker info | grep "Security Options"
# Should include: apparmor

# Install AppArmor utilities
sudo apt-get install apparmor-utils apparmor-profiles
```

## Step 1: View the Docker Default Profile

```bash
# The default Docker AppArmor profile is stored at:
cat /etc/apparmor.d/docker-default
# or
cat /etc/apparmor.d/containers/lxc-container-default

# Check which profile a container is using
docker inspect my_container --format '{{.AppArmorProfile}}'
# Returns: docker-default
```

## Step 2: Create a Custom AppArmor Profile

```bash
# /etc/apparmor.d/docker-nginx
# Custom AppArmor profile for Nginx containers

#include <tunables/global>

profile docker-nginx flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  # Allow network access
  network inet tcp,
  network inet6 tcp,
  network inet udp,
  network inet6 udp,

  # Deny raw network access (containers shouldn't need this)
  deny network raw,
  deny network packet,

  # Allow reading nginx config and static files
  /etc/nginx/** r,
  /var/www/** r,
  /usr/share/nginx/** r,

  # Allow writing to log and cache directories
  /var/log/nginx/** rw,
  /var/cache/nginx/** rw,
  /var/run/nginx.pid rw,

  # Allow nginx binary and libraries
  /usr/sbin/nginx mr,
  /usr/lib/** mr,
  /lib/** mr,

  # Allow proc and sys reads (needed for system info)
  @{PROC}/sys/kernel/ngroups_max r,
  @{PROC}/sys/net/core/somaxconn r,

  # Deny sensitive system paths
  deny /proc/sys/kernel/sysrq w,
  deny /sys/kernel/security/** rwklx,

  # Allow sending signals to child processes
  signal (send) set=(kill, term, usr1) peer=docker-nginx,

  # Deny ptrace (debugging other processes)
  deny ptrace,

  # Capabilities
  capability net_bind_service,  # Bind to ports < 1024
  capability setuid,            # Drop privileges
  capability setgid,
  capability dac_override,      # Read/write files regardless of owner
  deny capability sys_admin,
  deny capability sys_module,
  deny capability sys_ptrace,
}
```

## Step 3: Load and Verify the Profile

```bash
# Load the profile
sudo apparmor_parser -r -W /etc/apparmor.d/docker-nginx

# Verify it's loaded
sudo aa-status | grep docker-nginx
# Should show: docker-nginx (enforce)

# Load in complain mode first for testing (log violations, don't block)
sudo apparmor_parser -C /etc/apparmor.d/docker-nginx
# or
sudo aa-complain /etc/apparmor.d/docker-nginx

# Check violations in complain mode
sudo aa-logprof  # Interactive tool to review and allow violations
journalctl -k | grep apparmor | tail -20
```

## Step 4: Apply Profile to Containers via Portainer

```yaml
# docker-compose.yml - Custom AppArmor profile
version: "3.8"

services:
  nginx:
    image: nginx:alpine
    security_opt:
      # Apply custom profile
      - apparmor:docker-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - nginx_logs:/var/log/nginx

  api:
    image: myapp/api:latest
    security_opt:
      # Use Docker's default profile (explicit)
      - apparmor:docker-default

  # Unconfined (no AppArmor) - NOT recommended for production
  legacy_app:
    image: legacy:latest
    security_opt:
      - apparmor:unconfined

volumes:
  nginx_logs:
```

## Step 5: Profile for a Node.js Application

```bash
# /etc/apparmor.d/docker-nodejs-api

#include <tunables/global>

profile docker-nodejs-api flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  # Network access for HTTP server
  network inet tcp,
  network inet6 tcp,

  # Node.js binary
  /usr/local/bin/node mr,

  # Application files (read-only)
  /app/** r,
  /app/node_modules/** mr,

  # Allow writing to specific directories only
  /app/logs/** rw,
  /tmp/** rw,

  # Node.js needs access to these
  /proc/*/status r,
  /proc/*/maps r,

  # Deny writes to application code (immutable app)
  deny /app/*.js w,
  deny /app/package.json w,

  # Allow required libraries
  /usr/** mr,
  /lib/** mr,

  # Deny dangerous capabilities
  deny capability sys_admin,
  deny capability sys_module,
  deny capability mknod,
}
```

## Step 6: Enforce and Monitor

```bash
# Switch profile from complain to enforce mode
sudo aa-enforce /etc/apparmor.d/docker-nodejs-api

# Monitor AppArmor denials in real-time
sudo journalctl -k -f | grep "apparmor.*DENIED"

# If a container crashes after applying profile, check logs:
journalctl -k | grep apparmor | grep DENIED | tail -30
# Look for the denied operation and path, add it to profile if legitimate

# Reload profile after editing
sudo apparmor_parser -r /etc/apparmor.d/docker-nginx

# Verify container is using the profile
docker inspect nginx_container --format '{{.AppArmorProfile}}'
```

## Conclusion

AppArmor profiles complement seccomp profiles - AppArmor controls file system access and capabilities, while seccomp controls which system calls can be made. Together they provide defense-in-depth protection. Start with complain mode to discover what your container legitimately needs, review the logs, then enforce the profile in production. Docker's default profile is a good baseline, but a custom profile tailored to your specific application eliminates entire categories of potential exploit paths. Portainer's `security_opt` configuration makes it simple to deploy different AppArmor profiles per service in your stack YAML.
