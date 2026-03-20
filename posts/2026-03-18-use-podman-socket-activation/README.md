# How to Use Podman for Socket Activation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Socket Activation, Systemd, Linux, Containers

Description: Learn how to use systemd socket activation with Podman to start containers on demand, reducing resource usage and enabling zero-downtime container management.

---

> Socket activation lets you start containers only when they receive incoming connections, eliminating idle resource consumption. Combined with Podman's systemd integration, this creates efficient, on-demand container services.

Socket activation is a systemd feature that listens on a socket and starts the associated service only when a connection arrives. When paired with Podman, this means containers stay stopped until they are actually needed, saving CPU and memory. This guide walks you through setting up socket-activated Podman containers from scratch.

---

## How Socket Activation Works

The flow of socket activation with Podman follows these steps:

1. Systemd creates a socket and listens for connections
2. When a client connects, systemd starts the associated container
3. The socket file descriptor is passed to the container
4. The application inside the container accepts the connection
5. When the container exits, systemd continues listening

This pattern is useful for services that receive intermittent traffic, development environments, and reducing the footprint of microservice deployments.

## Prerequisites

Ensure you have Podman and systemd installed:

```bash
podman --version
systemctl --version
```

Socket activation works with both rootful and rootless Podman. This guide focuses on rootless operation for better security.

## Setting Up the Podman API Socket

Podman itself uses socket activation for its API service. Enable it:

```bash
# Enable the Podman API socket (rootless)

systemctl --user enable --now podman.socket

# Verify the socket is listening
systemctl --user status podman.socket

# Check the socket path
echo $XDG_RUNTIME_DIR/podman/podman.sock
```

Test the API through the socket:

```bash
curl --unix-socket $XDG_RUNTIME_DIR/podman/podman.sock \
  http://localhost/v4.0.0/libpod/info | python3 -m json.tool | head -20
```

## Creating a Socket-Activated Container Service

### Step 1: Create the Socket Unit

Create a systemd socket unit file:

```bash
mkdir -p ~/.config/systemd/user/
```

```ini
# ~/.config/systemd/user/my-web.socket
[Unit]
Description=My Web Server Socket

[Socket]
ListenStream=8080
Accept=no

[Install]
WantedBy=sockets.target
```

### Step 2: Create the Service Unit

Create the corresponding service unit that starts a Podman container:

```ini
# ~/.config/systemd/user/my-web.service
[Unit]
Description=My Web Server Container
Requires=my-web.socket
After=my-web.socket

[Service]
Type=notify
Environment=PODMAN_SYSTEMD_UNIT=%n
ExecStartPre=-/usr/bin/podman rm -f my-web
ExecStart=/usr/bin/podman run \
    --name my-web \
    --rm \
    --sdnotify=conmon \
    --replace \
    docker.io/library/nginx:latest
ExecStop=/usr/bin/podman stop my-web
TimeoutStartSec=90
Restart=on-failure

[Install]
WantedBy=default.target
```

### Step 3: Enable and Start

```bash
# Reload systemd to pick up new unit files
systemctl --user daemon-reload

# Enable the socket (not the service)
systemctl --user enable --now my-web.socket

# Verify the socket is listening
systemctl --user status my-web.socket
ss -tlnp | grep 8080
```

### Step 4: Test Socket Activation

The container is not running yet:

```bash
podman ps  # No my-web container
```

Make a request to trigger activation:

```bash
curl http://localhost:8080
```

Now the container is running:

```bash
podman ps  # my-web container is running
systemctl --user status my-web.service
```

## Socket Activation with Quadlet

Podman Quadlet provides a simpler way to define container services. Create Quadlet files in `~/.config/containers/systemd/`:

```bash
mkdir -p ~/.config/containers/systemd/
```

```ini
# ~/.config/containers/systemd/my-api.container
[Unit]
Description=My API Server

[Container]
Image=docker.io/library/python:3.11-slim
ContainerName=my-api
PublishPort=9090:8000
Exec=python3 -m http.server 8000
Environment=PYTHONUNBUFFERED=1

[Service]
Restart=on-failure
TimeoutStartSec=60

[Install]
WantedBy=default.target
```

Create the socket file:

```ini
# ~/.config/containers/systemd/my-api.socket
[Unit]
Description=My API Server Socket

[Socket]
ListenStream=9090
Accept=no

[Install]
WantedBy=sockets.target
```

Activate:

```bash
systemctl --user daemon-reload
systemctl --user enable --now my-api.socket
```

## Advanced Socket Configuration

### Multiple Listening Ports

```ini
# ~/.config/systemd/user/multi-port.socket
[Unit]
Description=Multi-Port Service Socket

[Socket]
ListenStream=8080
ListenStream=8443
ListenStream=/run/user/%U/my-service.sock

[Install]
WantedBy=sockets.target
```

### Rate Limiting

Prevent abuse by limiting connection rates:

```ini
# ~/.config/systemd/user/rate-limited.socket
[Unit]
Description=Rate Limited Service Socket

[Socket]
ListenStream=8080
Accept=no
MaxConnections=100
MaxConnectionsPerSource=10
TriggerLimitIntervalSec=2s
TriggerLimitBurst=20

[Install]
WantedBy=sockets.target
```

### Socket Permissions

Control who can connect:

```ini
[Socket]
ListenStream=8080
SocketUser=myuser
SocketGroup=mygroup
SocketMode=0660
```

## Generating Systemd Units from Containers

Note: The `podman generate systemd` command is deprecated as of Podman 4.4. Use Quadlet files (shown above) instead for new deployments. The command below still works but will not receive new features:

```bash
# Create and configure a container first
podman create --name my-service \
  -p 8080:80 \
  nginx:latest

# Generate systemd unit files (DEPRECATED - use Quadlet instead)
podman generate systemd --new --name my-service \
  --files \
  --restart-policy=on-failure
```

This creates service files that you can enhance with socket activation. For new projects, prefer writing Quadlet `.container` files directly.

## Monitoring Socket-Activated Services

### Checking Socket Status

```bash
# List all sockets
systemctl --user list-sockets

# Check specific socket status
systemctl --user status my-web.socket

# See socket connections
systemctl --user show my-web.socket | grep -i "n.*connections"
```

### Viewing Activation Logs

```bash
# See when the service was socket-activated
journalctl --user -u my-web.service --since "1 hour ago"

# See socket events
journalctl --user -u my-web.socket --since "1 hour ago"
```

### Automation Script for Monitoring

```python
#!/usr/bin/env python3
"""Monitor socket-activated Podman services."""

import subprocess
import json

def get_socket_status(unit_name):
    """Get the status of a systemd socket unit."""
    result = subprocess.run(
        ["systemctl", "--user", "show", unit_name, "--output=json"],
        capture_output=True, text=True
    )

    if result.returncode != 0:
        return None

    lines = result.stdout.strip().split("\n")
    status = {}
    for line in lines:
        if "=" in line:
            key, _, value = line.partition("=")
            status[key] = value

    return status

def monitor_sockets(socket_names):
    """Monitor multiple socket units."""
    for name in socket_names:
        status = get_socket_status(name)
        if status:
            active = status.get("ActiveState", "unknown")
            sub = status.get("SubState", "unknown")
            accepted = status.get("NAccepted", "0")
            connected = status.get("NConnections", "0")

            print(f"Socket: {name}")
            print(f"  State: {active} ({sub})")
            print(f"  Accepted: {accepted}")
            print(f"  Current connections: {connected}")
            print()

monitor_sockets(["my-web.socket", "my-api.socket"])
```

## Idle Timeout and Auto-Stop

Configure containers to stop after a period of inactivity:

```ini
# ~/.config/systemd/user/idle-web.service
[Unit]
Description=Auto-stopping Web Server
Requires=idle-web.socket

[Service]
Type=notify
ExecStart=/usr/bin/podman run \
    --name idle-web \
    --rm \
    --sdnotify=conmon \
    --replace \
    -p 8080:80 \
    nginx:latest
ExecStop=/usr/bin/podman stop idle-web
TimeoutStopSec=30
# Stop after 5 minutes of no connections
TimeoutIdleSec=300
Restart=no

[Install]
WantedBy=default.target
```

## Socket Activation for Development

Use socket activation to manage development services efficiently:

```bash
#!/bin/bash
# dev-services.sh - Set up socket-activated development services

SERVICES=(
    "postgres:5432:postgres:15"
    "redis:6379:redis:7"
    "mailhog:8025:mailhog/mailhog:latest"
)

for service_spec in "${SERVICES[@]}"; do
    IFS=':' read -r name port image <<< "$service_spec"

    # Create socket unit
    cat > ~/.config/systemd/user/dev-${name}.socket << EOF
[Unit]
Description=Dev ${name} Socket

[Socket]
ListenStream=${port}
Accept=no

[Install]
WantedBy=sockets.target
EOF

    # Create service unit
    cat > ~/.config/systemd/user/dev-${name}.service << EOF
[Unit]
Description=Dev ${name} Container
Requires=dev-${name}.socket

[Service]
Type=notify
ExecStart=/usr/bin/podman run --name dev-${name} --rm --replace --sdnotify=conmon -p ${port}:${port} ${image}
ExecStop=/usr/bin/podman stop dev-${name}
Restart=on-failure

[Install]
WantedBy=default.target
EOF

    echo "Created units for dev-${name} on port ${port}"
done

systemctl --user daemon-reload

for service_spec in "${SERVICES[@]}"; do
    IFS=':' read -r name port image <<< "$service_spec"
    systemctl --user enable --now dev-${name}.socket
    echo "Enabled dev-${name}.socket"
done
```

## Conclusion

Socket activation with Podman creates efficient, on-demand container services that only consume resources when they are actively handling requests. This approach is ideal for development environments with many services, microservice deployments with intermittent traffic, and systems where resource efficiency matters. Combined with Podman's rootless operation and Quadlet integration, socket activation provides a production-ready pattern for systemd-managed container services.
