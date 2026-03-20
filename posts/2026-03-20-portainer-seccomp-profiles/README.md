# How to Configure Seccomp Profiles for Containers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Seccomp, Container Security, Linux Security, Docker Hardening

Description: Learn how to apply seccomp profiles to Docker containers via Portainer to restrict the system calls available to container processes.

---

Seccomp (Secure Computing Mode) filters system calls at the kernel level. By applying a seccomp profile, you limit which syscalls a container process can make, reducing the attack surface significantly. Docker ships a default seccomp profile; Portainer lets you apply custom profiles per container.

## Understanding the Default Profile

Docker's default seccomp profile allows about 300 of the ~400 available system calls and blocks dangerous ones like `ptrace`, `process_vm_readv`, and `mount`:

```bash
# Check if the default seccomp profile is applied
docker inspect $(docker ps -qf name=my-container) | \
  jq '.[0].HostConfig.SecurityOpt'
# Returns: ["seccomp=<json-profile>"] or null (null uses default)
```

## Applying a Custom Seccomp Profile via CLI

```bash
docker run -d \
  --security-opt seccomp=/path/to/profile.json \
  --name my-app \
  my-api:latest
```

## Applying a Seccomp Profile in a Stack

Reference the profile file in the Compose security options:

```yaml
version: "3.8"

services:
  api:
    image: my-api:latest
    security_opt:
      - seccomp:/etc/docker/seccomp/api-profile.json
    # ... rest of config
```

The profile file must be present on the Docker host at the specified path.

## Creating a Minimal Seccomp Profile

Start with an allow-list approach — permit only the syscalls your application needs:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "accept4", "access", "arch_prctl", "bind", "brk", "capget",
        "capset", "chdir", "chmod", "chown", "clone", "close", "connect",
        "dup", "dup2", "dup3", "epoll_create1", "epoll_ctl", "epoll_wait",
        "execve", "exit", "exit_group", "faccessat", "fchmod", "fchown",
        "fcntl", "fstat", "fstatfs", "futex", "getcwd", "getdents64",
        "getegid", "geteuid", "getgid", "getpid", "getppid", "getuid",
        "ioctl", "kill", "lstat", "mmap", "mprotect", "munmap",
        "nanosleep", "newfstatat", "open", "openat", "pipe", "pipe2",
        "poll", "ppoll", "pread64", "prlimit64", "pwrite64", "read",
        "readlink", "readlinkat", "recvfrom", "recvmsg", "rename",
        "rt_sigaction", "rt_sigprocmask", "rt_sigreturn", "sched_yield",
        "select", "sendmsg", "sendto", "set_tid_address", "setgid",
        "setsockopt", "setuid", "sigaltstack", "socket", "stat",
        "statfs", "symlink", "tgkill", "umask", "uname", "unlink",
        "wait4", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

## Discovering Required Syscalls

Use `strace` to identify which syscalls your application makes:

```bash
# Run with seccomp disabled to capture all syscalls
docker run --rm \
  --security-opt seccomp=unconfined \
  my-api:latest \
  strace -f -c -e trace=all node server.js 2>&1 | tail -40
```

The `-c` flag summarizes syscall counts — focus on the most-called ones as your allow-list baseline.

## Applying via Portainer UI

For standalone containers:

1. Go to **Containers > Add container**.
2. Expand **Runtime & Resources > Security/Host**.
3. In **Security options**, add: `seccomp=/etc/docker/seccomp/myapp-profile.json`
4. The profile file must exist on the Docker host.

## Disabling Seccomp for Debugging

If a container fails due to a blocked syscall, temporarily disable seccomp to confirm:

```yaml
services:
  api:
    security_opt:
      - seccomp:unconfined    # Disables all seccomp filtering
```

Then check `dmesg` or `/var/log/audit/audit.log` for blocked syscalls:

```bash
sudo dmesg | grep -i "seccomp"
# Or if audit is enabled:
sudo ausearch -m SECCOMP | tail -20
```

Add any blocked syscalls to your allow-list, then re-enable the profile.
