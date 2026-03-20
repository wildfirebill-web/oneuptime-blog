# How to Configure Seccomp Profiles for Containers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Security, Seccomp, Container Hardening, Linux Security

Description: Apply seccomp profiles to restrict system calls available to containers, reducing attack surface by limiting what containerized processes can ask the kernel to do.

## Introduction

Seccomp (Secure Computing Mode) is a Linux kernel feature that restricts which system calls a process can make. Docker applies a default seccomp profile that blocks ~44 dangerous syscalls. Custom profiles let you restrict containers even further — a web server doesn't need `ptrace`, `mount`, or `keyctl`. Fewer allowed syscalls mean a smaller attack surface. This guide covers creating and applying custom seccomp profiles via Portainer.

## Step 1: Understanding the Default Profile

```bash
# Docker's default seccomp profile blocks syscalls like:
# - ptrace (debugging/tracing other processes)
# - mount/umount (filesystem manipulation)
# - kexec_load (loading new kernel)
# - keyctl (kernel key management)
# - add_key, request_key

# Check if a container is using the default profile
docker inspect my_container --format '{{.HostConfig.SecurityOpt}}'
# [] means default seccomp profile is applied

# View the full default profile (built into Docker)
docker run --rm --security-opt seccomp=unconfined \
  alpine strace -c sleep 1 2>&1 | head -20
```

## Step 2: Create a Restrictive Custom Profile

```json
// /etc/docker/seccomp/nginx-profile.json
// Start from Docker's default and add restrictions
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "accept4", "bind", "brk", "chdir",
        "clock_gettime", "close", "connect",
        "dup", "dup2", "epoll_create1", "epoll_ctl",
        "epoll_wait", "exit", "exit_group",
        "fcntl", "fstat", "futex", "getdents64",
        "getegid", "geteuid", "getgid", "getpid",
        "getppid", "getrandom", "gettid", "getuid",
        "ioctl", "listen", "lseek", "mmap",
        "mprotect", "munmap", "nanosleep", "newfstatat",
        "open", "openat", "pipe2", "poll",
        "prctl", "pread64", "read", "recv",
        "recvfrom", "recvmsg", "rt_sigaction",
        "rt_sigprocmask", "rt_sigreturn", "sched_yield",
        "send", "sendfile", "sendmsg", "sendto",
        "set_robust_list", "setgid", "setuid",
        "socket", "socketpair", "stat", "sysinfo",
        "umask", "uname", "unlink", "wait4", "write",
        "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

## Step 3: Apply Seccomp Profile in Docker Compose

```yaml
# docker-compose.yml - Custom seccomp profile
version: "3.8"

services:
  nginx:
    image: nginx:alpine
    security_opt:
      # Apply custom restrictive profile
      - seccomp:/etc/docker/seccomp/nginx-profile.json
    ports:
      - "80:80"

  api:
    image: myapp/api:latest
    security_opt:
      # Use Docker's default profile (explicit)
      - seccomp:/etc/docker/seccomp/default.json

  # Disable seccomp entirely (NOT recommended for production)
  debug_container:
    image: ubuntu:22.04
    security_opt:
      - seccomp:unconfined
    command: ["tail", "-f", "/dev/null"]
```

## Step 4: Generate a Profile from Container Activity

```bash
# Step 1: Run container with audit mode (log but don't block)
docker run -d \
  --security-opt seccomp=/etc/docker/seccomp/audit.json \
  --name nginx_audit \
  nginx:alpine

# audit.json - logs all syscalls
# {
#   "defaultAction": "SCMP_ACT_LOG",
#   "syscalls": []
# }

# Step 2: Collect syscall logs from kernel audit
ausearch -m SECCOMP | grep nginx | awk '{print $NF}' | sort -u

# Step 3: Use oci-seccomp-bpf-hook to auto-generate profiles
# Install: https://github.com/containers/oci-seccomp-bpf-hook
docker run --annotation io.containers.trace-syscall=/tmp/profile.json \
  nginx:alpine

# Step 4: The generated profile only allows observed syscalls
cat /tmp/profile.json
```

## Step 5: Profile for Node.js Application

```json
// node-api-seccomp.json - Generated for a Node.js HTTP API
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "accept4", "bind", "brk", "chdir", "chmod",
        "clock_getres", "clock_gettime", "clock_nanosleep",
        "close", "connect", "dup", "dup2", "dup3",
        "epoll_create1", "epoll_ctl", "epoll_pwait",
        "epoll_wait", "eventfd2", "execve", "exit",
        "exit_group", "faccessat", "fadvise64", "fallocate",
        "fchdir", "fchmod", "fchown", "fcntl",
        "fdatasync", "flock", "fstat", "fstatfs",
        "fsync", "ftruncate", "futex", "getdents64",
        "getegid", "geteuid", "getgid", "getgroups",
        "getpeername", "getpid", "getppid", "getrandom",
        "getrlimit", "getsockname", "getsockopt",
        "gettid", "getuid", "inotify_add_watch",
        "inotify_init1", "ioctl", "listen", "lseek",
        "lstat", "madvise", "mkdir", "mkdirat",
        "mmap", "mprotect", "munmap", "nanosleep",
        "newfstatat", "open", "openat", "pipe2",
        "poll", "ppoll", "prctl", "pread64",
        "prlimit64", "pwrite64", "read", "readlink",
        "readlinkat", "recv", "recvfrom", "recvmsg",
        "rename", "renameat", "rmdir", "rt_sigaction",
        "rt_sigprocmask", "rt_sigreturn", "rt_sigsuspend",
        "sched_getaffinity", "sched_yield", "send",
        "sendmsg", "sendto", "set_robust_list",
        "set_tid_address", "setgid", "setgroups",
        "setuid", "shutdown", "sigaltstack", "socket",
        "stat", "statfs", "symlink", "symlinkat",
        "tgkill", "umask", "uname", "unlink",
        "unlinkat", "utimensat", "wait4", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

## Step 6: Verify Profile is Active

```bash
# Check security options on running container
docker inspect nginx_secured --format '{{.HostConfig.SecurityOpt}}'
# Returns: [seccomp:/etc/docker/seccomp/nginx-profile.json]

# Test that blocked syscalls are denied
docker exec nginx_secured strace -c sleep 1
# strace itself requires ptrace - if blocked, shows permission denied

# Verify application still works normally
curl http://localhost:80
# Should work if allowed syscalls cover nginx's needs
```

## Conclusion

Custom seccomp profiles are the most granular syscall-level security control for containers. Start with Docker's default profile (already applied unless you opted out), then tighten further for specific workloads you understand well. A web server needs far fewer syscalls than a general-purpose system utility. The profile generation approach — audit then allowlist — is safer than hand-crafting allow lists. Portainer's `security_opt` field in stack configurations makes it straightforward to deploy containers with custom profiles at scale.
