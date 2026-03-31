# How to Use Docker Compose v2 with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Docker Compose v2, Compatibility

Description: Learn how to use the official Docker Compose v2 plugin with Podman through the Docker-compatible socket API.

---

> Docker Compose v2 works natively with Podman through the socket API, giving you the latest Compose features without Docker Desktop.

Docker Compose v2 is the Go-based rewrite of Docker Compose that runs as a Docker CLI plugin (`docker compose` instead of `docker-compose`). It can communicate with any Docker-compatible API, including the Podman socket, giving you access to the latest Compose features while using Podman as the runtime.

---

## Installing Docker Compose v2

```bash
# Download the Docker Compose v2 plugin

DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
mkdir -p $DOCKER_CONFIG/cli-plugins

# Download the latest release
curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o $DOCKER_CONFIG/cli-plugins/docker-compose
chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose

# Verify
docker compose version
```

## Connecting to Podman

```bash
# Enable the Podman socket
systemctl --user enable --now podman.socket

# Set DOCKER_HOST to the Podman socket
export DOCKER_HOST=unix:///run/user/$(id -u)/podman/podman.sock

# Verify the connection
docker info
# Should show Podman version info
```

## Running Compose v2 Commands

```bash
# All standard Compose v2 commands work
docker compose up -d
docker compose ps
docker compose logs -f
docker compose down
docker compose exec web sh
docker compose build
```

## Compose v2 Features with Podman

```yaml
# docker-compose.yml - using Compose v2 features
services:
  web:
    image: docker.io/library/nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      db:
        condition: service_healthy
        restart: true
    develop:
      watch:
        - action: sync
          path: ./html
          target: /usr/share/nginx/html
  db:
    image: docker.io/library/postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5
```

```bash
# Use Compose v2 features like watch
docker compose up -d
docker compose watch
```

## Compose v2 vs podman-compose

```bash
# Docker Compose v2 - official tool, more features, uses socket API
docker compose up -d

# podman-compose - Python-based, calls podman CLI directly
podman-compose up -d

# Both read the same docker-compose.yml format
```

## Using Docker CLI with Podman

```bash
# Install the Docker CLI (without Docker Engine)
# The CLI is just a client that talks to the API

# With DOCKER_HOST set, docker commands go to Podman
docker ps          # Shows Podman containers
docker images      # Shows Podman images
docker compose up  # Runs through Podman
```

## Shell Profile Configuration

```bash
# Add to ~/.bashrc or ~/.zshrc for persistent setup
cat >> ~/.bashrc << 'EOF'
# Use Podman as Docker backend
export DOCKER_HOST=unix:///run/user/$(id -u)/podman/podman.sock
EOF

source ~/.bashrc
```

## Troubleshooting

```bash
# Check the socket is running
systemctl --user status podman.socket

# Test the API
curl --unix-socket /run/user/$(id -u)/podman/podman.sock \
  http://localhost/_ping
# Should return: OK

# If compose fails, check Podman version
podman --version
# Docker Compose v2 works best with Podman 4.x+
```

## Summary

Docker Compose v2 works with Podman by connecting through the Docker-compatible socket API. Enable the Podman socket, set `DOCKER_HOST`, and use `docker compose` commands as usual. This gives you the latest Compose features including watch mode, improved dependency handling, and profiles, all running on Podman.
