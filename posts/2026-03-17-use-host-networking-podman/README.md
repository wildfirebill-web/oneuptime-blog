# How to Use Host Networking with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Networking, Host Network, Performance

Description: Learn how to use host networking mode in Podman to share the host's network namespace with containers.

---

> Host networking mode bypasses container network isolation, giving the container direct access to the host's network interfaces. This eliminates NAT overhead and provides maximum network performance.

In host networking mode, the container shares the host's network namespace. The container does not get its own IP address; instead, it binds directly to the host's network interfaces. This is useful for performance-sensitive applications and network monitoring tools.

---

## Using Host Networking

```bash
# Run a container with host networking
podman run -d --name fast-web \
  --network host \
  docker.io/library/nginx:latest

# The container binds directly to port 80 on the host
# No port mapping needed - the container uses host ports directly
curl http://localhost:80
```

## When to Use Host Networking

```bash
# Network monitoring tools need raw access to host interfaces
podman run --rm --network host \
  docker.io/library/alpine:latest ip addr show

# Performance-sensitive applications avoid NAT overhead
podman run -d --name perf-app \
  --network host \
  docker.io/library/nginx:latest

# Applications that bind to many ports
podman run -d --name multi-port \
  --network host \
  my-app-with-many-ports:latest
```

## Host Networking vs Bridge Networking

| Feature | Host Network | Bridge Network |
|---------|-------------|---------------|
| Port mapping | Not needed | Required (-p) |
| Network isolation | None | Full |
| Performance | Best (no NAT) | Good |
| Port conflicts | Possible | Avoided |
| DNS resolution | Host DNS | Container DNS |

## Verifying Host Network Mode

```bash
# Check that the container uses host networking
podman inspect fast-web --format '{{ .HostConfig.NetworkMode }}'
# Output: host

# The container sees the host's interfaces
podman exec fast-web ip addr show

# The container shares the host's hostname
podman exec fast-web hostname
```

## Port Conflicts with Host Networking

Since containers share the host's ports, you cannot run multiple containers on the same port:

```bash
# First container binds to port 80
podman run -d --name web1 --network host \
  docker.io/library/nginx:latest

# Second container will fail if it also tries port 80
podman run -d --name web2 --network host \
  docker.io/library/nginx:latest
# Error: port 80 already in use

# Solution: configure the application to use a different port
podman run -d --name web2 --network host \
  -e NGINX_PORT=8081 \
  docker.io/library/nginx:latest
```

## Host Networking in Rootless Mode

```bash
# Rootless host networking has limitations
# Containers cannot bind to privileged ports (< 1024) without configuration
podman run --rm --network host \
  docker.io/library/alpine:latest \
  sh -c "nc -l -p 8080 &; echo 'Listening on 8080'"

# Allow rootless binding to low ports
sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80
```

## Host Networking for Network Debugging

```bash
# Use host networking for network diagnostic tools
podman run --rm --network host \
  docker.io/library/alpine:latest \
  sh -c "apk add --no-cache tcpdump && tcpdump -i eth0 -c 10"

# Check host network configuration from a container
podman run --rm --network host \
  docker.io/library/alpine:latest \
  sh -c "apk add --no-cache iproute2 && ip route show"
```

## Security Considerations

Host networking removes network isolation. The container can:
- See all host network traffic
- Bind to any available port
- Access all host network interfaces

```bash
# For sensitive workloads, prefer bridge networks with port mapping
podman run -d --name secure-app \
  --network my-bridge \
  -p 8080:80 \
  docker.io/library/nginx:latest
```

## Summary

Host networking in Podman shares the host's network namespace with the container, providing direct access to host interfaces and eliminating NAT overhead. Use `--network host` for performance-sensitive applications, network monitoring tools, and multi-port services. Be aware of port conflicts between containers and the reduced security isolation. In rootless mode, low port binding requires system configuration changes.
