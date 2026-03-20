# How to Prevent Container Escape Attacks with Portainer Settings (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Security, Container Escape, Privilege Escalation, Hardening

Description: Harden Docker containers against escape attacks by disabling privileged mode, restricting mounts, enabling seccomp, and applying the no-new-privileges flag via Portainer.

## Introduction

Container escape attacks allow a process inside a container to break out to the host system, gaining full root access. Common vectors include privileged container abuse, dangerous mounts (like `/var/run/docker.sock`), kernel exploit paths via unrestricted syscalls, and setuid binary escalation. This guide covers specific configurations in Portainer that close the most common escape paths.

## Step 1: Never Use Privileged Mode

Privileged containers are equivalent to root on the host:

```yaml
# DANGEROUS - allows container escape

services:
  api:
    privileged: true  # Never do this unless absolutely required

# SECURE - disabled privileged mode (default)
services:
  api:
    image: myapp/api:latest
    # No 'privileged: true' - this is the safe default

# If you see 'privileged: true' in a compose file, question it.
# Most services claiming to need it don't actually need it.
# Alternatives:
#   - Use specific cap_add instead of full privilege
#   - Use device mappings for hardware access
#   - Use specific mounts instead of full host access
```

```bash
# Audit for privileged containers
docker ps -q | xargs docker inspect --format \
  '{{.Name}}: privileged={{.HostConfig.Privileged}}' | grep "true"
# Any output here is a security concern
```

## Step 2: Block Privilege Escalation

```yaml
# docker-compose.yml - Prevent setuid/setgid escalation
version: "3.8"

services:
  api:
    image: myapp/api:latest

    # Prevent gaining new privileges via setuid binaries
    security_opt:
      - no-new-privileges:true

    # Run as non-root user
    user: "1000:1000"

    # Explicitly drop capabilities (belt and suspenders)
    cap_drop:
      - ALL

    # Even if attacker finds a setuid binary inside the container,
    # no-new-privileges prevents it from elevating privileges
```

## Step 3: Secure or Avoid Docker Socket Mounts

Mounting the Docker socket inside a container gives it full Docker daemon access:

```yaml
# DANGEROUS - container can create new privileged containers
services:
  app:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # Avoid this

# If you MUST mount the socket (e.g., Portainer agent, Watchtower):
services:
  portainer_agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro  # Read-only
      # ro prevents writing but still allows read of sensitive data
    # Consider using Portainer's TLS-based agent instead

# Alternatives to Docker socket mounting:
# 1. Use Docker TCP API with TLS authentication
# 2. Use docker-proxy tools that restrict API access
# 3. Use Podman's rootless daemon socket
```

## Step 4: Restrict Host Filesystem Mounts

```yaml
version: "3.8"

services:
  api:
    image: myapp/api:latest
    volumes:
      # SAFE: named volumes (isolated from host)
      - app_data:/app/data

      # SAFE: specific config file (read-only)
      - ./config.yml:/app/config.yml:ro

      # DANGEROUS: host path mounts (especially these):
      # - /etc:/etc              (host config)
      # - /proc:/host-proc       (kernel info)
      # - /sys:/sys              (kernel interface)
      # - /dev:/dev              (all devices)
      # - /boot:/boot            (kernel/bootloader)
      # - /var/lib/docker:/docker (container storage)

volumes:
  app_data:
```

## Step 5: Seccomp to Block Kernel Exploit Vectors

```json
// /etc/docker/seccomp/hardened.json
// Block syscalls commonly used in container escape exploits
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": [
        "accept4", "bind", "brk", "chdir", "clock_gettime",
        "close", "connect", "dup", "dup2", "epoll_create1",
        "epoll_ctl", "epoll_wait", "exit", "exit_group",
        "fcntl", "fstat", "futex", "getdents64", "getegid",
        "geteuid", "getgid", "getpid", "getppid", "getrandom",
        "gettid", "getuid", "ioctl", "listen", "lseek",
        "mmap", "mprotect", "munmap", "nanosleep", "newfstatat",
        "open", "openat", "poll", "read", "recv", "recvfrom",
        "rt_sigaction", "rt_sigprocmask", "rt_sigreturn",
        "send", "sendto", "set_robust_list", "setgid", "setuid",
        "socket", "stat", "uname", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
    // Blocked: ptrace, mount, umount, kexec_load, keyctl,
    // add_key, request_key, pivot_root, clone (with CLONE_NEWUSER)
  ]
}
```

```yaml
services:
  api:
    security_opt:
      - seccomp:/etc/docker/seccomp/hardened.json
      - no-new-privileges:true
```

## Step 6: User Namespace Remapping

User namespace remapping maps root inside the container to an unprivileged user on the host:

```json
// /etc/docker/daemon.json - Enable user namespace remapping
{
  "userns-remap": "default"
}
```

```bash
# Apply daemon change
sudo systemctl restart docker

# Now root (UID 0) inside container maps to high UID on host
# Even if container escape happens, attacker is unprivileged on host

# Verify
docker run --rm alpine id
# Inside container: uid=0(root) gid=0(root)
# But on host, this process runs as uid=165536 (unprivileged)
ps aux | grep alpine
# Shows host uid=165536
```

## Step 7: Runtime Security with Falco

```yaml
# docker-compose.yml - Falco runtime threat detection
version: "3.8"

services:
  falco:
    image: falcosecurity/falco-no-driver:latest
    container_name: falco
    restart: unless-stopped
    privileged: true  # Falco itself needs kernel access
    volumes:
      - /var/run/docker.sock:/host/var/run/docker.sock:ro
      - /dev:/host/dev:ro
      - /proc:/host/proc:ro
      - /boot:/host/boot:ro
      - /lib/modules:/host/lib/modules:ro
      - /usr:/host/usr:ro
      - /etc/falco:/etc/falco
    # Falco detects container escapes in real-time:
    # - Shell spawned in container
    # - Sensitive file reads (/etc/shadow, /etc/passwd)
    # - Unexpected outbound connections
    # - Privilege escalation attempts
```

## Conclusion

Container escape prevention is achieved through defense-in-depth: no privileged mode, no dangerous mounts, capability dropping, `no-new-privileges`, seccomp filtering, and optionally user namespace remapping. Each layer independently blocks different attack vectors. Regular audits using `docker inspect` to check for privileged mode and socket mounts are essential - these settings can drift as teams add new services. Portainer's stack YAML is the authoritative source of truth; reviewing it in code reviews catches security regressions before deployment.
