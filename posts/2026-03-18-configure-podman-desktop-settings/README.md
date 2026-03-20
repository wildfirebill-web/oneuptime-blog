# How to Configure Podman Desktop Settings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Podman Desktop, Configuration, Settings

Description: Learn how to configure Podman Desktop settings including registries, proxies, resources, and extensions for an optimized container workflow.

---

> Properly configuring Podman Desktop settings ensures your container development environment matches your infrastructure requirements and personal workflow preferences.

Podman Desktop comes with sensible defaults, but most teams need to customize settings for their specific environment. This includes configuring registries, adjusting resource limits, setting up proxy connections, and managing extensions. This post walks through the key settings in Podman Desktop and how to configure them for different scenarios.

---

## Accessing Settings

Open Podman Desktop settings through the application menu or the gear icon.

```bash
# Launch Podman Desktop

# On macOS:
open -a "Podman Desktop"

# On Linux (Flatpak):
flatpak run io.podman_desktop.PodmanDesktop

# Navigate to: Settings (gear icon in the bottom-left sidebar)
```

The Settings page is organized into sections: Resources, Registries, Proxy, and Extensions.

## Configuring Container Engine Resources

On macOS and Windows, Podman runs inside a virtual machine. You can adjust the VM resources.

```bash
# From the CLI, stop the current machine
podman machine stop

# Remove the existing machine
podman machine rm

# Create a new machine with custom resources
podman machine init \
    --cpus 4 \
    --memory 8192 \
    --disk-size 100

# Start the machine
podman machine start

# Verify the resource settings
podman machine inspect | python3 -m json.tool
```

In Podman Desktop, navigate to Settings > Resources to see the current Podman machine and its configuration. You can start, stop, and delete machines from this interface.

## Configuring Registries

Add container registries for pulling and pushing images.

```bash
# From the CLI, log in to a registry
podman login registry.example.com

# Log in to Docker Hub
podman login docker.io

# Log in to GitHub Container Registry
echo "$GITHUB_TOKEN" | podman login ghcr.io -u "$GITHUB_USER" --password-stdin

# Verify configured registries
podman info | grep -A 10 registries
```

In Podman Desktop, go to Settings > Registries to add registry credentials through the GUI. Click "Add Registry" and enter the server URL, username, and password.

## Configuring Registry Search Order

Set which registries are searched when you use short image names.

```bash
# Edit the registries configuration file
# On Linux:
sudo nano /etc/containers/registries.conf

# On macOS (inside the Podman machine):
podman machine ssh
sudo vi /etc/containers/registries.conf
```

Add or modify the unqualified search registries:

```toml
# /etc/containers/registries.conf
unqualified-search-registries = ["docker.io", "quay.io", "ghcr.io"]
```

```bash
# Verify the search order
podman info | grep -A 5 "search"
```

## Configuring Proxy Settings

If your network requires a proxy, configure it in Podman Desktop.

```bash
# Set proxy environment variables for the Podman machine
podman machine ssh

# Inside the machine, edit the environment
sudo tee /etc/systemd/system/podman.service.d/proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:8080"
Environment="HTTPS_PROXY=http://proxy.example.com:8080"
Environment="NO_PROXY=localhost,127.0.0.1,.internal"
EOF

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart podman
exit
```

In Podman Desktop, navigate to Settings > Proxy to configure proxy settings through the GUI.

## Configuring Extensions

Podman Desktop supports extensions for additional functionality.

```bash
# List installed extensions from the Podman Desktop GUI
# Navigate to Settings > Extensions

# Common extensions include:
# - Kubernetes (for managing K8s clusters)
# - Compose (for Docker Compose support)
# - Kind (for local Kubernetes clusters)
# - Podman AI Lab (for AI/ML workloads)
```

To install an extension, go to Settings > Extensions, browse the catalog, and click Install on the extension you want.

## Configuring Docker Compatibility

Enable Docker API compatibility for tools that expect Docker.

```bash
# Stop and reconfigure the Podman machine for rootful mode
podman machine stop
podman machine set --rootful

# Start the machine
podman machine start

# Verify the Docker socket is available
podman machine inspect | grep -i socket

# Test with a Docker-compatible command
curl --unix-socket /var/run/docker.sock http://localhost/version 2>/dev/null | python3 -m json.tool
```

In Podman Desktop, the Docker Compatibility section under Settings shows the socket status and lets you toggle compatibility mode.

## Configuring Default Container Settings

Set default values for container creation.

```bash
# Configure default container settings via containers.conf
# On Linux:
nano ~/.config/containers/containers.conf

# On macOS (inside the Podman machine):
podman machine ssh
vi /etc/containers/containers.conf
```

Example configuration:

```toml
[containers]
# Default environment variables for all containers
env = ["TZ=UTC"]

# Default DNS servers
dns_servers = ["8.8.8.8", "8.8.4.4"]

# Default logging driver
log_driver = "journald"

# Default ulimits
default_ulimits = [
    "nofile=1024:1024"
]
```

```bash
# Verify the configuration
podman info | grep -A 20 "store"
```

## Configuring Storage

Adjust storage settings for images and containers.

```bash
# View current storage configuration
podman info | grep -A 10 "store"

# Edit storage configuration
# On Linux:
nano ~/.config/containers/storage.conf

# Example: change the storage driver or location
# [storage]
# driver = "overlay"
# graphroot = "/var/lib/containers/storage"
```

## Summary

Podman Desktop settings let you customize your container environment for your specific needs. Key configuration areas include machine resources (CPU, memory, disk), registry credentials and search order, proxy settings for corporate networks, extensions for additional features, and Docker compatibility mode. Most settings can be configured through both the Podman Desktop GUI and CLI configuration files. Taking time to configure these settings properly ensures a smooth container development workflow tailored to your infrastructure.
