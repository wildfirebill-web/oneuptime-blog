# How to Use x-podman Extensions in Compose Files

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, podman-compose, Extensions, Configuration

Description: Learn how to use x-podman extension fields in Compose files to access Podman-specific features like pods and rootful mode.

---

> x-podman extensions let you use Podman-specific features in your compose files while keeping them compatible with Docker Compose.

The Compose specification allows custom extension fields prefixed with `x-`. podman-compose recognizes `x-podman` extensions to expose Podman-specific capabilities like pod grouping, rootful containers, and custom Podman arguments that Docker Compose does not support natively.

---

## Grouping Services into a Pod

```yaml
# docker-compose.yml
version: "3.8"
services:
  web:
    image: docker.io/library/nginx:alpine
    ports:
      - "8080:80"
    x-podman:
      # Place this service in a shared pod
      in_pod: true
  api:
    image: docker.io/library/python:3.12-slim
    command: python -m http.server 5000
    x-podman:
      in_pod: true
```

```bash
# Start services — web and api share a pod
podman-compose up -d

# Verify the pod was created
podman pod ls
```

## Running as Rootful

```yaml
# docker-compose.yml
version: "3.8"
services:
  privileged-app:
    image: docker.io/library/alpine:latest
    command: sleep infinity
    x-podman:
      # Run this container with elevated privileges
      rootful: true
```

## Custom Podman Run Arguments

```yaml
# docker-compose.yml
version: "3.8"
services:
  app:
    image: docker.io/library/nginx:alpine
    x-podman:
      # Pass additional arguments to podman run
      podman_args:
        - "--userns=keep-id"
        - "--security-opt=label=disable"
```

## Specifying Container Names

```yaml
# docker-compose.yml
version: "3.8"
services:
  db:
    image: docker.io/library/postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    x-podman:
      # Override the auto-generated container name
      container_name: my-database
```

## Using No-Hosts Option

```yaml
# docker-compose.yml
version: "3.8"
services:
  app:
    image: docker.io/library/alpine:latest
    command: sleep infinity
    x-podman:
      # Do not add host entries to /etc/hosts
      no_hosts: true
```

## Combining Multiple Extensions

```yaml
# docker-compose.yml
version: "3.8"
x-podman:
  # Global Podman settings
  default_infra_name: myproject-infra

services:
  web:
    image: docker.io/library/nginx:alpine
    ports:
      - "8080:80"
    x-podman:
      in_pod: true
      podman_args:
        - "--cpus=1.0"
        - "--memory=512m"
  worker:
    image: docker.io/library/python:3.12-slim
    command: python -c "import time; time.sleep(3600)"
    x-podman:
      in_pod: true
      podman_args:
        - "--cpus=0.5"
        - "--memory=256m"
```

```bash
# Deploy with all Podman-specific options
podman-compose up -d

# Verify resource limits
podman inspect web --format '{{.HostConfig.NanoCpus}}'
```

## Docker Compose Compatibility

```bash
# x-podman fields are ignored by Docker Compose
# so the same file works with both tools
docker compose up -d  # Ignores x-podman
podman-compose up -d  # Uses x-podman extensions
```

## Summary

Use `x-podman` extension fields in your compose files to access Podman-specific features like pod grouping, rootful mode, and custom run arguments. These extensions are ignored by Docker Compose, so your compose files remain compatible with both tools.
