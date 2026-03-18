# How to Use Podman with VS Code Dev Containers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, VS Code, Dev Containers, Docker Alternative, Development Environments

Description: Learn how to configure Visual Studio Code Dev Containers to work with Podman instead of Docker, enabling rootless container-based development environments.

---

> Podman offers a rootless, daemonless alternative to Docker that integrates seamlessly with VS Code Dev Containers once you know how to set it up.

VS Code Dev Containers let you define your entire development environment as code, spinning up isolated containers with all the tools, runtimes, and dependencies your project needs. While Docker has been the default container runtime for this workflow, Podman is an increasingly popular alternative. Podman runs without a background daemon, supports rootless containers out of the box, and maintains CLI compatibility with Docker. This post walks you through configuring Podman as the container engine for VS Code Dev Containers on Linux, macOS, and Windows.

---

## Prerequisites

Before getting started, make sure you have the following installed on your system:

- Visual Studio Code (1.74 or later)
- The Dev Containers extension (`ms-vscode-remote.remote-containers`)
- Podman (4.0 or later)

On macOS and Windows, Podman runs inside a lightweight Linux virtual machine. You need to initialize and start that machine before VS Code can connect.

```bash
# Install Podman on macOS via Homebrew
brew install podman

# Initialize the Podman machine (creates a Linux VM)
podman machine init

# Start the Podman machine
podman machine start

# Verify Podman is running
podman info
```

On Linux, Podman runs natively without a VM:

```bash
# Install Podman on Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y podman

# Verify the installation
podman --version
```

## Configuring VS Code to Use Podman

VS Code Dev Containers defaults to Docker as its container runtime. You need to update your VS Code settings to point it at Podman instead. Open your VS Code settings (JSON) and add the following:

```json
{
  // Tell Dev Containers to use Podman instead of Docker
  "dev.containers.dockerPath": "podman",

  // Use Podman Compose if you need multi-container setups
  "dev.containers.dockerComposePath": "podman-compose"
}
```

These two settings redirect all container commands from `docker` to `podman`. Since Podman is CLI-compatible with Docker, most commands work without modification.

## Setting Up the Docker Socket Compatibility

Some VS Code extensions and Dev Container features expect a Docker socket at `/var/run/docker.sock`. On macOS and Windows, enable the Podman socket with the following commands:

```bash
# Enable the Podman socket on macOS/Windows
# This creates a Docker-compatible API endpoint
podman machine stop
podman machine set --rootful
podman machine start

# Verify the socket is available
podman system connection list
```

On Linux, you can enable the Podman socket as a systemd user service:

```bash
# Enable and start the Podman socket service (rootless)
systemctl --user enable --now podman.socket

# Verify the socket path
echo $XDG_RUNTIME_DIR/podman/podman.sock
```

You can optionally create a symlink so tools expecting the Docker socket find Podman instead:

```bash
# Create a symlink for Docker socket compatibility on Linux
sudo ln -sf /run/user/$(id -u)/podman/podman.sock /var/run/docker.sock
```

## Creating a Dev Container Configuration

With Podman configured, create a `.devcontainer` directory in your project and add a `devcontainer.json` file. Here is an example for a Node.js project:

```json
{
  "name": "Node.js Dev Environment",
  // Use an official Node.js image
  "image": "node:20-bookworm",

  // Features to install inside the container
  "features": {
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },

  // Forward port 3000 for local development
  "forwardPorts": [3000],

  // Run npm install after the container is created
  "postCreateCommand": "npm install",

  // Set the default shell
  "customizations": {
    "vscode": {
      "settings": {
        "terminal.integrated.defaultProfile.linux": "bash"
      },
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode"
      ]
    }
  }
}
```

Open the VS Code Command Palette (`Ctrl+Shift+P` or `Cmd+Shift+P`) and run **Dev Containers: Reopen in Container**. VS Code will use Podman to pull the image, create the container, and connect to it.

## Using a Containerfile (Dockerfile) with Podman

For more control over your environment, use a `Containerfile` (Podman's name for a Dockerfile). Create `.devcontainer/Containerfile`:

```dockerfile
# Start from a base image
FROM python:3.12-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install Python development tools
RUN pip install --no-cache-dir \
    pylint \
    black \
    pytest \
    debugpy

# Create a non-root user for development
ARG USERNAME=developer
ARG USER_UID=1000
ARG USER_GID=$USER_UID

RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME

USER $USERNAME
```

Then reference it in your `devcontainer.json`:

```json
{
  "name": "Python Dev Environment",
  // Build from the local Containerfile
  "build": {
    "dockerfile": "Containerfile",
    "context": ".."
  },
  "remoteUser": "developer",
  "forwardPorts": [8000],
  "postCreateCommand": "pip install -r requirements.txt"
}
```

## Multi-Container Development with Podman Compose

For projects that need multiple services (such as an application server and a database), you can use `podman-compose`. Install it first:

```bash
# Install podman-compose
pip install podman-compose
```

Create a `docker-compose.yml` file in your `.devcontainer` directory:

```yaml
version: "3.8"
services:
  app:
    build:
      context: ..
      dockerfile: .devcontainer/Containerfile
    volumes:
      - ..:/workspace:cached
    command: sleep infinity
    ports:
      - "3000:3000"

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Update `devcontainer.json` to use the compose file:

```json
{
  "name": "Full Stack Environment",
  "dockerComposeFile": "docker-compose.yml",
  "service": "app",
  "workspaceFolder": "/workspace",
  "shutdownAction": "stopCompose"
}
```

## Troubleshooting Common Issues

Working with Podman and Dev Containers can surface a few common issues. Here are solutions for the most frequent ones.

**Permission errors with mounted volumes:**

Podman's rootless mode maps user IDs differently than Docker. If you see permission errors on mounted files, set the `:Z` volume label or configure UID mapping:

```bash
# Check the current UID mapping
podman unshare cat /proc/self/uid_map

# If needed, add userns_mode to your devcontainer.json
# "runArgs": ["--userns=keep-id"]
```

In your `devcontainer.json`, add the `--userns=keep-id` flag:

```json
{
  "runArgs": ["--userns=keep-id"],
  "containerUser": "developer",
  "remoteUser": "developer"
}
```

**Container fails to start on macOS:**

Make sure the Podman machine is running and has enough resources:

```bash
# Check the machine status
podman machine info

# Increase resources if needed
podman machine stop
podman machine set --cpus 4 --memory 8192
podman machine start
```

**Extensions cannot connect to the container engine:**

Set the `DOCKER_HOST` environment variable to point at the Podman socket:

```bash
# On macOS, find the socket path
export DOCKER_HOST="unix://$(podman machine inspect --format '{{.ConnectionInfo.PodmanSocket.Path}}')"

# On Linux
export DOCKER_HOST="unix://$XDG_RUNTIME_DIR/podman/podman.sock"
```

## Conclusion

Podman is a capable drop-in replacement for Docker when used with VS Code Dev Containers. The key steps are pointing VS Code at the `podman` binary, enabling the Podman socket for API compatibility, and adding `--userns=keep-id` to handle rootless UID mapping. Once configured, you get the same containerized development experience with the added benefits of rootless execution and no background daemon. If you run into edge cases, the Podman and VS Code Dev Containers GitHub repositories both have active issue trackers where you can find solutions.
