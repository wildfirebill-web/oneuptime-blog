# How to Use Portainer with Podman as a Docker Alternative

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Podman, Docker, Containers, Linux

Description: Learn how to set up Portainer to manage containers running on Podman as a daemonless, rootless Docker alternative.

## Introduction

Podman is a daemonless container engine developed by Red Hat that offers a Docker-compatible CLI while providing enhanced security through rootless containers. Portainer can connect to Podman by exposing a Docker-compatible socket, giving you a full GUI for managing Podman containers.

## Prerequisites

- A Linux host (RHEL, Fedora, Ubuntu, or Debian)
- Podman installed (version 3.0+)
- Portainer CE or Business Edition

## Installing Podman

```bash
# Install Podman on RHEL/Fedora
sudo dnf install -y podman

# Install Podman on Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y podman

# Verify installation
podman --version
```

## Enabling the Podman Socket

Podman includes a systemd socket that exposes a Docker-compatible REST API. This is the key to integrating with Portainer.

```bash
# Enable and start the Podman socket for the current user (rootless)
systemctl --user enable --now podman.socket

# Verify the socket is active
systemctl --user status podman.socket

# Check the socket path
echo $XDG_RUNTIME_DIR/podman/podman.sock
# Typically: /run/user/1000/podman/podman.sock
```

For system-wide (root) socket:

```bash
# Enable the system-level Podman socket
sudo systemctl enable --now podman.socket

# Verify
sudo systemctl status podman.socket
# Socket path: /run/podman/podman.sock
```

## Deploying Portainer via Podman

```bash
# Create a volume for Portainer data
podman volume create portainer_data

# Run Portainer container using the Podman socket
podman run -d \
  --name portainer \
  --restart=always \
  -p 8000:8000 \
  -p 9443:9443 \
  -v /run/user/1000/podman/podman.sock:/var/run/docker.sock:Z \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Check that Portainer is running
podman ps
```

Note: The `:Z` label sets the SELinux context correctly for Podman volumes.

## Configuring Portainer to Use the Podman Socket

Once Portainer is running, navigate to `https://localhost:9443` and complete setup.

When adding an environment, select **Docker Standalone** and point it to the Podman socket:

```bash
# Unix socket path for local Podman
unix:///run/user/1000/podman/podman.sock

# Or for system socket
unix:///run/podman/podman.sock
```

## Creating a Podman Systemd Service for Portainer

For production deployments, create a systemd service:

```bash
# Generate systemd unit from running container
podman generate systemd --new --name portainer > ~/.config/systemd/user/portainer.service

# Reload systemd and enable the service
systemctl --user daemon-reload
systemctl --user enable --now portainer.service

# Check the service status
systemctl --user status portainer.service
```

## Handling SELinux with Podman

Podman runs SELinux enforcing by default on RHEL/Fedora. Use the `:Z` or `:z` volume labels:

```bash
# :Z creates a private label (only this container)
# :z creates a shared label (multiple containers can access)
podman run -d \
  --name my-app \
  -v /host/data:/container/data:Z \
  nginx:latest
```

## Verifying the Integration

After setup, you can manage Podman containers through Portainer just like Docker:

```bash
# List containers via Podman CLI
podman ps -a

# The same containers appear in Portainer's UI
# Navigate to: Environments > Local > Containers
```

## Differences to Be Aware Of

| Feature | Docker | Podman |
|---------|--------|--------|
| Daemon | Required | Daemonless |
| Root | Optional | Optional (rootless by default) |
| Compose | docker-compose | podman-compose or docker-compose |
| Pods | No | Yes (like Kubernetes pods) |
| Socket | /var/run/docker.sock | /run/user/UID/podman/podman.sock |

## Conclusion

Portainer provides a convenient web interface for managing Podman containers. By exposing the Podman socket, you get the full Portainer experience—stacks, volumes, networks, and more—without requiring the Docker daemon. This setup is ideal for security-focused environments where rootless containers and SELinux integration are requirements.
