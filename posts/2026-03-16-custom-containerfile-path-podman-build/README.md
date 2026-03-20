# How to Specify a Custom Containerfile Path with podman build

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Container Images, Build, Containerfile, Custom Path

Description: Learn how to use the -f flag with podman build to specify custom Containerfile paths for flexible project structures and multi-environment builds.

---

> Custom Containerfile paths let you organize builds for multiple environments without cluttering your project root.

Projects often require different container configurations for development, staging, and production. Instead of maintaining a single Containerfile in the project root, you can keep environment-specific Containerfiles in organized directories and reference them with the `-f` flag. This guide covers all the patterns for specifying custom Containerfile paths with Podman.

---

## The -f Flag

The `-f` or `--file` flag tells Podman where to find the Containerfile. The build context remains the directory you specify as the final argument.

```bash
# Basic syntax

podman build -f /path/to/Containerfile -t image:tag BUILD_CONTEXT
```

## Using a Containerfile from a Different Directory

```bash
# Project structure:
# myproject/
#   build/
#     Containerfile.dev
#     Containerfile.prod
#   src/
#     app.py
#   requirements.txt

# Build using the development Containerfile
podman build -f build/Containerfile.dev -t myapp:dev .

# Build using the production Containerfile
podman build -f build/Containerfile.prod -t myapp:prod .
```

The build context (`.`) is still the project root, so COPY instructions in the Containerfile can reference files relative to the project root.

## Environment-Specific Containerfiles

Create separate Containerfiles for each environment.

```bash
mkdir -p ~/myproject/docker

# Development Containerfile with debug tools
cat > ~/myproject/docker/Containerfile.dev << 'EOF'
FROM docker.io/library/python:3.12

WORKDIR /app

# Install development dependencies
COPY requirements.txt requirements-dev.txt ./
RUN pip install -r requirements.txt -r requirements-dev.txt

# Install debugging tools
RUN pip install debugpy ipdb

COPY . .

# Development server with hot reload
CMD ["python", "-m", "flask", "run", "--debug", "--host=0.0.0.0"]
EOF

# Production Containerfile, slim and optimized
cat > ~/myproject/docker/Containerfile.prod << 'EOF'
FROM docker.io/library/python:3.12-slim

WORKDIR /app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd -m appuser
USER appuser

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
EOF

# Build each environment
cd ~/myproject
podman build -f docker/Containerfile.dev -t myapp:dev .
podman build -f docker/Containerfile.prod -t myapp:prod .
```

## Using Absolute Paths

You can use absolute paths to reference Containerfiles anywhere on the filesystem.

```bash
# Build using an absolute path to the Containerfile
podman build -f /home/user/shared-configs/Containerfile.base -t base-image:latest .

# The build context is still the current directory
podman build -f /opt/containerfiles/Containerfile.nginx -t custom-nginx:latest /path/to/webroot
```

## Separating Build Context from Containerfile Location

The build context and Containerfile location are independent. This is powerful for monorepo setups.

```bash
# Monorepo structure:
# monorepo/
#   containerfiles/
#     Containerfile.api
#     Containerfile.frontend
#   services/
#     api/
#       src/
#       package.json
#     frontend/
#       src/
#       package.json

# Build the API service
# Containerfile is in containerfiles/, context is services/api/
podman build \
  -f containerfiles/Containerfile.api \
  -t myorg/api:latest \
  services/api/

# Build the frontend service
podman build \
  -f containerfiles/Containerfile.frontend \
  -t myorg/frontend:latest \
  services/frontend/
```

## Using Stdin as the Containerfile

You can pass a Containerfile via stdin, useful for dynamic or generated builds.

```bash
# Build from stdin using a heredoc
podman build -t quick-image:latest -f - . << 'EOF'
FROM docker.io/library/alpine:latest
RUN apk add --no-cache curl
CMD ["sh"]
EOF

# Generate a Containerfile dynamically
echo "FROM docker.io/library/nginx:latest" | podman build -t dynamic:latest -f - .
```

## Naming Conventions

Common naming patterns for organizing multiple Containerfiles.

```bash
# Pattern 1: Dot-separated suffix
# Containerfile.dev, Containerfile.prod, Containerfile.test

# Pattern 2: Directory-based organization
# docker/dev/Containerfile
# docker/prod/Containerfile
# docker/test/Containerfile

# Pattern 3: Descriptive names
# Containerfile-builder
# Containerfile-runtime
# Containerfile-debug

# Build with each pattern
podman build -f Containerfile.prod -t myapp:prod .
podman build -f docker/prod/Containerfile -t myapp:prod .
podman build -f Containerfile-runtime -t myapp:latest .
```

## CI/CD Integration

In CI/CD pipelines, custom Containerfile paths keep build definitions organized.

```bash
# CI/CD script example
#!/bin/bash
# build.sh - Build script for CI/CD

ENVIRONMENT="${1:-dev}"
TAG="${2:-latest}"
CONTAINERFILE="docker/Containerfile.${ENVIRONMENT}"

# Verify the Containerfile exists
if [ ! -f "$CONTAINERFILE" ]; then
  echo "Error: $CONTAINERFILE not found"
  exit 1
fi

echo "Building ${ENVIRONMENT} image with tag ${TAG}"
podman build \
  -f "$CONTAINERFILE" \
  -t "myapp:${TAG}" \
  --build-arg BUILD_ENV="${ENVIRONMENT}" \
  .

echo "Build complete: myapp:${TAG}"
```

```bash
# Usage:
# ./build.sh dev latest
# ./build.sh prod v1.2.3
# ./build.sh test ci-build-42
```

## Verifying the Correct File Is Used

Confirm that Podman is using the Containerfile you intended.

```bash
# Use --log-level to see which file Podman reads
podman build --log-level debug -f docker/Containerfile.prod -t myapp:prod . 2>&1 | head -20

# Add a comment in each Containerfile to identify it
# First line of Containerfile.prod:
# # Containerfile: production build
```

## Summary

The `-f` flag in `podman build` provides flexibility to organize Containerfiles for different environments, services, and build targets. Use it to maintain clean project structures with environment-specific builds, monorepo service builds, and CI/CD pipeline configurations. Always keep the build context path in mind since COPY instructions are relative to the build context, not the Containerfile location.
