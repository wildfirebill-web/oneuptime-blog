# How to Use Init Containers in a Podman Pod

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Pods, Init Containers, Startup

Description: Learn how to use init containers in Podman pods to run setup tasks before the main application starts.

---

> Init containers run to completion before the main containers start, handling setup tasks like database migrations and config generation.

Init containers are short-lived containers that perform initialization work before the main application containers begin. They run sequentially and must exit successfully before the next init container or the main containers start. This pattern is borrowed from Kubernetes and is available in Podman pods.

---

## Creating a Pod with an Init Container

```bash
# Create a pod
podman pod create --name app-pod -p 8080:80

# Run an init container that sets up configuration
podman run --pod app-pod --init-ctr always --name init-config \
  docker.io/library/alpine \
  sh -c "echo 'server_name=myapp' > /tmp/shared/config.ini"
```

The `--init-ctr` flag marks the container as an init container. The value `always` means it runs every time the pod starts.

## Init Container Types

```bash
# 'always' - runs every time the pod starts
podman run --pod app-pod --init-ctr always --name setup \
  docker.io/library/alpine echo "Running setup"

# 'once' - runs only on the first pod start
podman run --pod app-pod --init-ctr once --name first-run \
  docker.io/library/alpine echo "First time initialization"
```

## Use Case: Database Migration

```bash
# Create a pod for a web application
podman pod create --name web-pod -p 5000:5000

# Start the database first
podman run -d --pod web-pod --name db \
  -e POSTGRES_PASSWORD=secret \
  docker.io/library/postgres:16-alpine

# Run an init container that waits for the database and runs migrations
podman run --pod web-pod --init-ctr always --name migrate \
  docker.io/library/alpine \
  sh -c "
    echo 'Waiting for database...'
    sleep 5
    echo 'Running migrations...'
    echo 'Migrations complete'
  "

# The main application starts after migrations complete
podman run -d --pod web-pod --name app docker.io/library/alpine \
  sh -c "echo 'App starting after init' && sleep 3600"
```

## Use Case: Downloading Configuration

```bash
# Init container that fetches configuration before the app starts
podman pod create --name config-pod -v shared-data:/data

podman run --pod config-pod --init-ctr always --name fetch-config \
  -v shared-data:/data \
  docker.io/library/alpine \
  sh -c "echo '{\"setting\": \"value\"}' > /data/config.json && echo 'Config downloaded'"

# Main container uses the downloaded configuration
podman run -d --pod config-pod --name app \
  -v shared-data:/data \
  docker.io/library/alpine \
  sh -c "cat /data/config.json && sleep 3600"
```

## Verifying Init Container Execution

```bash
# List all containers including init containers
podman ps -a --filter pod=app-pod --format "table {{.Names}}\t{{.Status}}"

# Init containers show as Exited with code 0 on success
# Check init container logs
podman logs init-config
```

## Handling Init Container Failures

```bash
# If an init container exits with a non-zero code, the pod will not start
podman run --pod app-pod --init-ctr always --name bad-init \
  docker.io/library/alpine sh -c "echo 'failing' && exit 1"

# Check the exit code
podman inspect bad-init --format '{{.State.ExitCode}}'
# Output: 1
```

## Summary

Init containers in Podman pods run setup tasks before the main application starts. Use `--init-ctr always` for tasks that should run on every pod start and `--init-ctr once` for one-time initialization. Init containers are ideal for database migrations, configuration downloads, and dependency checks.
