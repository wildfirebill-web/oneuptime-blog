# How to Expose Container Ports and Bind to Specific IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Networking, IPv4, Port Binding, Containers, Security

Description: Publish Docker container ports and bind them to specific host IPv4 addresses to control which interfaces accept external connections, limiting exposure to trusted networks.

## Introduction

By default, `docker run -p 8080:80` binds port 8080 to all host interfaces (`0.0.0.0`), making the service accessible from anywhere. Binding to a specific host IPv4 address restricts access to only connections arriving on that interface — essential for security and multi-homed servers.

## Bind to All Interfaces (Default)

```bash
# Container port 80 accessible on all host interfaces at port 8080
docker run -d -p 8080:80 nginx:alpine

# Equivalent explicit form
docker run -d -p 0.0.0.0:8080:80 nginx:alpine
```

## Bind to a Specific Host IP

```bash
# Only accessible from the local machine (loopback)
docker run -d -p 127.0.0.1:8080:80 nginx:alpine

# Only accessible from the LAN interface
docker run -d -p 192.168.1.100:8080:80 nginx:alpine

# Only accessible from a specific management interface
docker run -d -p 10.0.0.5:8080:80 nginx:alpine
```

## Binding Multiple Ports to Different IPs

```bash
# Public-facing API on one IP, admin on another
docker run -d \
  -p 203.0.113.10:443:443 \
  -p 192.168.1.100:8443:8443 \
  my-app:latest
```

## Verifying the Binding

```bash
# Confirm which address:port the container is bound to
docker port nginx-container

# Output:
# 80/tcp -> 127.0.0.1:8080

# Or use ss on the host
ss -tlnp | grep 8080
```

## Docker Compose Port Binding

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      # Bind to loopback only
      - "127.0.0.1:8080:80"

  api:
    image: my-api:latest
    ports:
      # Bind to specific LAN IP
      - "192.168.1.100:3000:3000"

  db-admin:
    image: pgadmin4:latest
    ports:
      # Only accessible via management network
      - "10.0.0.5:5050:80"
```

## Setting Default Binding in daemon.json

To change the default binding address for all containers:

```bash
sudo tee -a /etc/docker/daemon.json << 'EOF'
{
  "ip": "127.0.0.1"
}
EOF
sudo systemctl restart docker
```

Now all `-p` without an explicit IP bind to `127.0.0.1` instead of `0.0.0.0`.

## Security Recommendation

| Scenario | Recommended Binding |
|---|---|
| Internal service (DB, cache) | `127.0.0.1:<port>` or not exposed |
| LAN-accessible service | `<LAN IP>:<port>` |
| Public web service | `0.0.0.0:<port>` (behind firewall) |
| Admin/debug interface | `127.0.0.1:<port>` |

## Conclusion

Use `<host-ip>:<host-port>:<container-port>` syntax to bind published ports to specific IPv4 interfaces. Bind admin and internal services to `127.0.0.1` for the strictest isolation. Set `"ip": "127.0.0.1"` in `daemon.json` to make localhost-only the default for all containers.
