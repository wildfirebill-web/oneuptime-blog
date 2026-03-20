# How to Configure docker-compose to Use Podman Backend

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Docker Compose, Backend, Configuration

Description: Learn how to configure docker-compose to use Podman as its backend runtime instead of Docker, including context setup and environment configuration.

---

> Switching docker-compose to use Podman as its backend lets your team keep existing Compose workflows while dropping the Docker daemon dependency.

Docker Compose can work with any OCI-compatible runtime that exposes a Docker-compatible API. By configuring it to use Podman as the backend, you get rootless containers, no daemon requirement, and full Compose compatibility in a single setup.

---

## Method 1: DOCKER_HOST Environment Variable

The simplest approach points Docker Compose at the Podman socket.

```bash
# Enable the Podman socket

systemctl --user enable --now podman.socket

# Set DOCKER_HOST globally
export DOCKER_HOST=unix:///run/user/$(id -u)/podman/podman.sock

# Test the connection
docker compose version
docker compose ls
```

## Method 2: Docker Context

Create a Docker context that points to Podman.

```bash
# Create a new context for Podman
docker context create podman \
  --docker "host=unix:///run/user/$(id -u)/podman/podman.sock"

# Switch to the Podman context
docker context use podman

# Verify the active context
docker context ls
# Output shows podman context with an asterisk

# Now all docker compose commands use Podman
docker compose up -d
```

## Method 3: Docker CLI Configuration File

```bash
# Create or edit the Docker CLI config
mkdir -p ~/.docker
cat > ~/.docker/config.json << 'EOF'
{
  "currentContext": "podman"
}
EOF
```

## Verifying the Backend

```bash
# Check which runtime is being used
docker info | grep -i "operating system\|server version"

# The output should reference Podman
# Server Version: 4.x.x (Podman)
```

## Running Compose Commands

```bash
# All standard docker compose commands work
docker compose up -d
docker compose ps
docker compose logs -f
docker compose down
```

## Example Compose File

```yaml
# docker-compose.yml
version: "3.8"
services:
  app:
    image: docker.io/library/node:20-alpine
    working_dir: /app
    volumes:
      - .:/app
    command: node server.js
    ports:
      - "3000:3000"
  db:
    image: docker.io/library/postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
volumes:
  db-data:
```

```bash
# Works identically with the Podman backend
docker compose up -d
docker compose ps
```

## Switching Between Docker and Podman

```bash
# List available contexts
docker context ls

# Switch to Podman
docker context use podman

# Switch back to Docker
docker context use default
```

## Persisting the Configuration

```bash
# Add to your shell profile
cat >> ~/.bashrc << 'EOF'
# Use Podman as Docker backend
export DOCKER_HOST=unix:///run/user/$(id -u)/podman/podman.sock
EOF

source ~/.bashrc
```

## Summary

Configure docker-compose to use Podman by setting `DOCKER_HOST` to the Podman socket, creating a Docker context, or editing the Docker CLI config file. All three methods let you run standard Docker Compose commands with Podman as the backend runtime, and you can switch between Docker and Podman contexts as needed.
