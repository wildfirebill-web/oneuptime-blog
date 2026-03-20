# How to Run Portainer with Read-Only Root Filesystem

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Read-Only Filesystem, Container Hardening, Docker Security

Description: Learn how to run Docker containers with a read-only root filesystem via Portainer to prevent malicious writes and reduce the attack surface.

---

Running containers with a read-only root filesystem prevents any process inside the container from writing to the filesystem layer — including malware, accidental config overwrites, and exploitation payloads. Portainer supports this via the `read_only` flag or the UI.

## Enabling Read-Only Root Filesystem in a Stack

Set `read_only: true` in your Compose service definition:

```yaml
version: "3.8"

services:
  api:
    image: my-api:latest
    read_only: true
    tmpfs:
      - /tmp:size=64m,mode=1777    # Writable temp directory in RAM
      - /run:size=10m              # For PID files and sockets
    volumes:
      - api_data:/app/data         # Named volume for persistent writes
    environment:
      NODE_ENV: production

volumes:
  api_data:
```

The `tmpfs` mounts provide writable scratch space in memory — essential for applications that need to write temporary files.

## Enabling via Portainer UI

For standalone containers:

1. Go to **Containers > Add container**.
2. Expand **Runtime & Resources**.
3. Enable **Read-only root filesystem**.
4. Add tmpfs mounts under **Volumes > Bind** with type `tmpfs` for `/tmp` and `/run`.

## Testing Which Paths Fail

Before enabling read-only mode, identify all paths the application writes to:

```bash
# Run the container with strace to find all write calls
docker run --rm \
  --security-opt seccomp=unconfined \
  my-api:latest \
  strace -f -e trace=write,openat -s 256 node server.js 2>&1 | \
  grep -v "^strace:" | grep -v ENOENT | head -50

# Or use inotifywait to monitor writes at runtime
docker exec -it my-api-container \
  sh -c "apt-get install -y inotify-tools && inotifywait -r -m / 2>/dev/null" &
```

## Common Application Requirements

| Application | Required Writable Paths |
|-------------|-------------------------|
| Node.js | `/tmp`, `/app/node_modules/.cache` |
| Python | `/tmp`, `/__pycache__` |
| Nginx | `/var/cache/nginx`, `/var/run`, `/tmp` |
| PHP-FPM | `/var/run`, `/tmp` |
| Java | `/tmp` |

## Nginx with Read-Only Filesystem

Nginx requires several writable directories. Configure tmpfs for each:

```yaml
services:
  nginx:
    image: nginx:alpine
    read_only: true
    tmpfs:
      - /var/cache/nginx:size=64m
      - /var/run:size=10m
      - /tmp:size=32m
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./html:/usr/share/nginx/html:ro
```

## Verifying Read-Only Enforcement

Confirm the filesystem is read-only:

```bash
# Try writing to the root filesystem — should fail
docker exec -it $(docker ps -qf name=api) \
  sh -c "echo test > /test.txt && echo 'FAIL: writable' || echo 'PASS: read-only'"

# Try writing to tmpfs — should succeed
docker exec -it $(docker ps -qf name=api) \
  sh -c "echo test > /tmp/test.txt && echo 'PASS: tmpfs writable' || echo 'FAIL'"
```

## Read-Only for Portainer Itself

Run Portainer with a read-only root filesystem:

```bash
docker run -d \
  --name portainer \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /run \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```
