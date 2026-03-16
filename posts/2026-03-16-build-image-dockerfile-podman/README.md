# How to Build an Image from a Dockerfile with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Container Images, Build, Dockerfile

Description: Learn how to build container images from Dockerfiles using Podman, with full compatibility for existing Docker-based workflows.

---

> Podman builds images from Dockerfiles with full compatibility, making migration from Docker seamless.

Podman fully supports the Dockerfile format, so you can use your existing Dockerfiles without modification. This makes Podman a drop-in replacement for Docker when building images. This guide covers building images from Dockerfiles with Podman, including compatibility details and practical examples.

---

## Dockerfile Support in Podman

Podman recognizes both `Dockerfile` and `Containerfile`. When both exist in the same directory, `Containerfile` takes priority. The syntax is identical.

```bash
# Podman file detection priority:
# 1. Containerfile
# 2. Dockerfile

# Check which file Podman will use
ls -la Containerfile Dockerfile 2>/dev/null
```

## Building from an Existing Dockerfile

If you have a project with a Dockerfile, Podman builds it directly.

```bash
# Create a sample project with a Dockerfile
mkdir -p ~/docker-project && cd ~/docker-project

cat > Dockerfile << 'EOF'
FROM docker.io/library/node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --production

COPY . .

EXPOSE 3000

CMD ["node", "index.js"]
EOF

# Create a simple application
cat > package.json << 'EOF'
{
  "name": "myapp",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "express": "^4.18.0"
  }
}
EOF

cat > index.js << 'EOF'
const express = require('express');
const app = express();
app.get('/', (req, res) => res.send('Hello from Podman'));
app.listen(3000, () => console.log('Server running on port 3000'));
EOF

# Build the image using the Dockerfile
podman build -t mynode-app:latest .
```

## Explicitly Specifying the Dockerfile

When your Dockerfile has a non-standard name or is in a different location, use the `-f` flag.

```bash
# Build from a specific Dockerfile
podman build -f Dockerfile -t myapp:latest .

# Build from a Dockerfile with a custom name
podman build -f Dockerfile.production -t myapp:prod .

# Build from a Dockerfile in another directory
podman build -f /path/to/Dockerfile -t myapp:latest .
```

## Docker CLI Compatibility

Podman supports the same build flags as Docker, making migration straightforward.

```bash
# These commands work identically in both Docker and Podman:

# Build with a tag
podman build -t myimage:latest .

# Build with multiple tags
podman build -t myimage:latest -t myimage:v1.0 .

# Build with build arguments
podman build --build-arg NODE_ENV=production -t myapp:latest .

# Build with no cache
podman build --no-cache -t myapp:latest .

# Build and show output
podman build --progress=plain -t myapp:latest .
```

## Docker-Specific Instructions

Podman supports all Docker-specific Dockerfile instructions.

```bash
cat > Dockerfile << 'EOF'
FROM docker.io/library/python:3.12-slim

# SHELL instruction (Docker-specific)
SHELL ["/bin/bash", "-c"]

# STOPSIGNAL instruction
STOPSIGNAL SIGTERM

# HEALTHCHECK instruction
HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

# ONBUILD instruction
ONBUILD COPY . /app

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

CMD ["python", "app.py"]
EOF

podman build -t py-app:latest .
```

## Migrating Docker Compose Builds

If you have a Docker Compose setup with build instructions, you can build the images individually with Podman.

```bash
# Example docker-compose.yml build section:
# services:
#   web:
#     build:
#       context: ./web
#       dockerfile: Dockerfile
#       args:
#         NODE_ENV: production

# Equivalent Podman build command
podman build \
  --build-arg NODE_ENV=production \
  -f ./web/Dockerfile \
  -t myapp-web:latest \
  ./web
```

## Creating a podman Alias for Docker

For teams migrating from Docker, create an alias so existing scripts work unchanged.

```bash
# Add to your shell profile (~/.bashrc or ~/.zshrc)
alias docker=podman

# Verify the alias works
docker build -t test:latest .
# (This runs podman build)

# For scripts that use the docker socket
export DOCKER_HOST=unix:///run/user/$(id -u)/podman/podman.sock
```

## Handling Docker Build Features

Some Docker build features have slightly different behavior in Podman.

```bash
# Docker BuildKit secrets (supported via --secret flag)
podman build --secret id=mysecret,src=secret.txt -t myapp:latest .

# Docker BuildKit SSH forwarding
podman build --ssh default -t myapp:latest .

# BuildKit inline cache (format flag may be needed)
podman build --format docker -t myapp:latest .
```

## Building a Multi-Service Application

Build multiple services from their respective Dockerfiles.

```bash
# Project structure:
# myproject/
#   frontend/
#     Dockerfile
#   backend/
#     Dockerfile
#   database/
#     Dockerfile

# Build all services
podman build -t myproject-frontend:latest ./frontend/
podman build -t myproject-backend:latest ./backend/
podman build -t myproject-database:latest ./database/

# Verify all images
podman images | grep myproject
```

## Testing the Built Image

Always verify your image works correctly after building.

```bash
# Run the image
podman run -d --name test -p 3000:3000 mynode-app:latest

# Check container logs
podman logs test

# Test the application
curl http://localhost:3000

# Check container health
podman inspect test --format '{{.State.Status}}'

# Clean up
podman stop test && podman rm test
```

## Summary

Podman provides full Dockerfile compatibility, making it a seamless replacement for Docker when building images. All standard Dockerfile instructions, build arguments, and build flags work identically. For teams migrating from Docker, the transition requires no changes to existing Dockerfiles. Simply replace `docker build` with `podman build` and your existing workflows continue to work.
