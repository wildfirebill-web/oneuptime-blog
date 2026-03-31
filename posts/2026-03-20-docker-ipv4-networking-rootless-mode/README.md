# How to Configure Docker Container IPv4 Networking in Rootless Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Networking, IPv4, Rootless, Security, Container

Description: Configure IPv4 networking for Docker containers running in rootless mode using slirp4netns or pasta, understand port binding limitations, and expose services without root privileges.

## Introduction

Docker rootless mode runs the Docker daemon as a non-root user, improving security by preventing container escapes from escalating to root. Networking in rootless mode uses a userspace network stack (`slirp4netns` or `pasta`) instead of kernel bridges, with different capabilities and limitations.

## Installing Docker Rootless

```bash
# Install rootless setup script dependencies

sudo apt install -y uidmap dbus-user-session

# Run the rootless install script
curl -fsSL https://get.docker.com/rootless | sh

# Start the rootless daemon
systemctl --user start docker
systemctl --user enable docker

# Set environment variables
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```

## Network Behavior in Rootless Mode

Rootless Docker uses `slirp4netns` (or the newer `pasta`) for container networking:
- Containers are NATed through the user's network stack
- No kernel bridge is created
- No `docker0` interface visible in `ip link show`
- Containers can still reach the internet via NAT

```bash
# Confirm network mode
docker info 2>/dev/null | grep -i "cgroup\|network"

# Containers still get IPs from the default range
docker run --rm alpine ip addr show
```

## Exposing Ports in Rootless Mode

Non-root users cannot bind to ports below 1024 by default. Ports 80, 443 require either:
- Kernel capability: `net_bind_service`
- `sysctl net.ipv4.ip_unprivileged_port_start=0` (allows all users to bind to any port)
- Binding to ports ≥ 1024 and using a reverse proxy

```bash
# Allow rootless Docker to bind to privileged ports
echo "net.ipv4.ip_unprivileged_port_start=0" | sudo tee -a /etc/sysctl.d/99-rootless.conf
sudo sysctl --system

# Now rootless containers can bind to port 80
docker run -d -p 80:80 nginx:alpine
```

## Using pasta Instead of slirp4netns

`pasta` (Pack A Subtle Tap Abstraction) is a faster alternative to `slirp4netns`:

```bash
# Install pasta
sudo apt install pasta

# Configure Docker rootless to use pasta
mkdir -p ~/.config/docker
tee ~/.config/docker/daemon.json << 'EOF'
{
  "network": {
    "driver": "bridge",
    "pasta": true
  }
}
EOF

systemctl --user restart docker
```

## Custom Network Configuration in Rootless Mode

```bash
# Create a custom network - works the same as rootful Docker
docker network create --subnet 192.168.200.0/24 mynet

# Run containers on the custom network
docker run -d --network mynet --name web nginx:alpine
docker run -d --network mynet --name db postgres:15-alpine
```

## Limitations of Rootless Networking

| Feature | Rootful | Rootless |
|---|---|---|
| Ports < 1024 | Yes (default) | Requires sysctl tuning |
| macvlan networks | Yes | No (requires root) |
| IPVLAN networks | Yes | No |
| Host network mode | Yes | Limited |
| Custom bridge networks | Yes | Yes |

## Conclusion

Docker rootless mode supports standard bridge networking and container-to-container communication through userspace NAT. Enable low-port binding with `net.ipv4.ip_unprivileged_port_start=0` for web services. macvlan and ipvlan networks are not available in rootless mode.
