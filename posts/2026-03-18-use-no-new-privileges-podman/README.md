# How to Use No-New-Privileges with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Linux, Security, Hardening, Privileges

Description: Learn how to use the no-new-privileges flag in Podman to prevent container processes from gaining additional privileges through setuid binaries or other escalation mechanisms.

---

> No-new-privileges is a simple kernel-level guarantee that a process can never gain more privileges than its parent.

The `no-new-privileges` flag is a Linux kernel feature that prevents a process and its children from ever acquiring new privileges. When enabled, setuid and setgid binaries cannot escalate permissions, and the process cannot gain capabilities beyond what it started with. Podman supports this flag as a critical hardening measure for containers.

This guide explains how no-new-privileges works, how to enable it in Podman, and why it should be part of your container security baseline.

---

## How No-New-Privileges Works

The `no_new_privs` bit is a process attribute in the Linux kernel. Once set, it applies to the current process and all its descendants and cannot be unset.

```bash
# Check if no_new_privs is set inside a default container
# A value of 0 means it is not set; 1 means it is active
podman run --rm docker.io/library/alpine:latest \
  sh -c "cat /proc/self/status | grep NoNewPrivs"
```

When `no_new_privs` is set:
- Setuid and setgid bits on executables are ignored
- The process cannot call `execve()` to gain new capabilities
- Child processes inherit the restriction

## Enabling No-New-Privileges

Use the `--security-opt no-new-privileges` flag when running a container.

```bash
# Run a container with no-new-privileges enabled
podman run --rm \
  --security-opt no-new-privileges \
  docker.io/library/alpine:latest \
  sh -c "cat /proc/self/status | grep NoNewPrivs"

# The output should show NoNewPrivs: 1
```

## Demonstrating Setuid Protection

The primary benefit of no-new-privileges is blocking setuid-based privilege escalation.

```bash
# Without no-new-privileges, setuid binaries can escalate
# The ping command typically has setuid or capabilities
podman run --rm \
  docker.io/library/alpine:latest \
  sh -c "ls -la /bin/ping 2>/dev/null; ping -c 1 127.0.0.1"

# With no-new-privileges, setuid binaries cannot escalate
# This blocks one of the most common privilege escalation vectors
podman run --rm \
  --security-opt no-new-privileges \
  docker.io/library/alpine:latest \
  sh -c "ping -c 1 127.0.0.1 2>&1 || echo 'ping may fail without privilege escalation'"
```

## Combining with Capability Restrictions

No-new-privileges works best when combined with dropped capabilities for defense in depth.

```bash
# Drop all capabilities and enable no-new-privileges
# This creates a heavily restricted container environment
podman run --rm \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  docker.io/library/alpine:latest \
  sh -c "
    echo 'Capabilities:' && \
    cat /proc/self/status | grep -E 'Cap|NoNewPrivs'
  "

# Add back only needed capabilities while keeping no-new-privileges
podman run --rm \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  docker.io/library/alpine:latest \
  sh -c "cat /proc/self/status | grep -E 'Cap|NoNewPrivs'"
```

## Practical Example: Hardened Web Server

Apply no-new-privileges to a production web server container.

```bash
# Run nginx with no-new-privileges and minimal capabilities
podman run -d \
  --name hardened-nginx \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  -p 8080:80 \
  docker.io/library/nginx:alpine

# Verify the security settings
podman inspect hardened-nginx --format '{{.HostConfig.SecurityOpt}}'

# Confirm no-new-privileges is active inside the container
podman exec hardened-nginx \
  sh -c "cat /proc/self/status | grep NoNewPrivs"

# Test that the web server still works
curl -s http://localhost:8080 | head -3

# Clean up
podman stop hardened-nginx && podman rm hardened-nginx
```

## Using with Non-Root Containers

No-new-privileges is especially important for containers that run as non-root.

```bash
# Run as a non-root user with no-new-privileges
# Even if a setuid binary exists, it cannot escalate to root
podman run --rm \
  --user 1000:1000 \
  --security-opt no-new-privileges \
  docker.io/library/alpine:latest \
  sh -c "
    echo 'Running as:' && id
    echo 'NoNewPrivs:' && cat /proc/self/status | grep NoNewPrivs
  "
```

## Configuring in Podman Compose

Apply no-new-privileges across all services in a compose file.

```bash
# Create a compose file with no-new-privileges for all services
cat > /tmp/hardened-compose.yml << 'EOF'
version: "3"
services:
  web:
    image: docker.io/library/nginx:alpine
    ports:
      - "8080:80"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
  app:
    image: docker.io/library/python:3-slim
    command: python3 -m http.server 8000
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
EOF

echo "Hardened compose file created at /tmp/hardened-compose.yml"
```

## Summary

The `--security-opt no-new-privileges` flag is one of the most effective container hardening measures available. It prevents all forms of privilege escalation through setuid binaries, setgid binaries, and capability transitions. Enable it on every production container, combine it with dropped capabilities and non-root users, and consider making it the default in your Podman configuration. The performance overhead is zero, and the security benefit is substantial.
