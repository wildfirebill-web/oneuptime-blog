# How to Run a Container with Retry Options in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Reliability, Retry Options

Description: Learn how to configure retry options in Podman to handle transient image pull failures and improve container startup reliability.

---

> Retry options protect your container workflows from transient network failures during image pulls, making deployments more resilient.

Network issues can cause image pulls to fail intermittently, especially in CI/CD pipelines or cloud environments. Podman provides retry options that automatically reattempt failed pulls, reducing the impact of transient errors. This guide covers how to configure retry behavior for reliable container operations.

---

## Understanding Retry Options

Podman supports two retry-related flags:

- `--retry <count>` - Number of times to retry a failed image pull (default: 3)
- `--retry-delay <duration>` - Time to wait between retry attempts (default: exponential backoff starting at 2 seconds)

These flags apply to the image pull operation that occurs when you run a container and the image is not available locally.

## Basic Retry Configuration

```bash
# Retry up to 5 times if the image pull fails

podman run --rm \
  --retry 5 \
  docker.io/library/alpine:latest \
  echo "Image pulled successfully"
```

## Setting Retry with Delay

Adding a delay between retries gives the network time to recover.

```bash
# Retry 3 times with a 10-second delay between attempts
podman run --rm \
  --retry 3 \
  --retry-delay 10s \
  docker.io/library/alpine:latest \
  echo "Pull succeeded after retries"
```

Supported duration formats include seconds (`s`), minutes (`m`), and combinations:

```bash
# Retry with a 30-second delay
podman run --rm \
  --retry 5 \
  --retry-delay 30s \
  docker.io/library/nginx:latest \
  echo "Nginx ready"

# Retry with a 1-minute delay
podman run --rm \
  --retry 3 \
  --retry-delay 1m \
  docker.io/library/postgres:16 \
  echo "Postgres ready"
```

## Retry Options with podman pull

The retry flags also work with the standalone `podman pull` command.

```bash
# Pull an image with retry options
podman pull \
  --retry 5 \
  --retry-delay 15s \
  docker.io/library/alpine:latest

# Then run the container without needing retries
podman run --rm docker.io/library/alpine:latest echo "Running from cache"
```

## CI/CD Pipeline Usage

Retry options are especially valuable in automated environments.

```bash
#!/bin/bash
# ci-build.sh - CI pipeline with robust image pulling

IMAGE="docker.io/library/node:20-alpine"

echo "Starting CI build..."

# Use aggressive retry settings for CI environments
podman run --rm \
  --retry 5 \
  --retry-delay 15s \
  -v ./:/app:z \
  -w /app \
  "$IMAGE" \
  sh -c 'npm ci && npm test'

EXIT_CODE=$?

if [ $EXIT_CODE -eq 0 ]; then
  echo "CI build passed"
else
  echo "CI build failed with exit code $EXIT_CODE"
  exit $EXIT_CODE
fi
```

## Retry with Multiple Images

When working with multiple images, apply retry options to each operation.

```bash
#!/bin/bash
# pull-all.sh - Pull multiple images with retry

IMAGES=(
  "docker.io/library/alpine:latest"
  "docker.io/library/nginx:latest"
  "docker.io/library/postgres:16"
  "docker.io/library/redis:7"
)

for IMAGE in "${IMAGES[@]}"; do
  echo "Pulling $IMAGE..."
  podman pull --retry 3 --retry-delay 10s "$IMAGE"
  if [ $? -eq 0 ]; then
    echo "Successfully pulled $IMAGE"
  else
    echo "Failed to pull $IMAGE after retries"
    exit 1
  fi
done

echo "All images pulled successfully"
```

## Retry Options with podman create

Retry settings work with `podman create` as well.

```bash
# Create a container with retry options (does not start it)
podman create \
  --name my-app \
  --retry 5 \
  --retry-delay 20s \
  docker.io/library/alpine:latest \
  echo "Container created"

# Start it later
podman start my-app
```

## Disabling Retries

Set retry to 0 to disable automatic retries.

```bash
# No retries - fail immediately on pull error
podman run --rm \
  --retry 0 \
  docker.io/library/alpine:latest \
  echo "No retries"
```

## Combining Retry with Pull Policy

Retry options interact with pull policies.

```bash
# Always pull with retries (ideal for CI/CD)
podman run --rm \
  --pull=always \
  --retry 5 \
  --retry-delay 10s \
  docker.io/library/alpine:latest \
  echo "Fresh image with reliable pull"

# Never pull - retries are irrelevant since no pull occurs
podman run --rm \
  --pull=never \
  docker.io/library/alpine:latest \
  echo "Local only, no retries needed"
```

## Configuring Default Retry in containers.conf

Set default retry behavior in your Podman configuration.

```bash
# Edit containers.conf for default retry settings
mkdir -p ~/.config/containers
cat >> ~/.config/containers/containers.conf << 'EOF'
[engine]
retry = 3
retry_delay = "10s"
EOF

# Now all pulls use these defaults unless overridden
podman pull docker.io/library/alpine:latest
```

## Summary

Retry options in Podman provide resilience against transient network failures during image pulls. Use `--retry` to set the number of attempts and `--retry-delay` to add a pause between retries. Configure defaults in `containers.conf` for consistent behavior across all operations. These settings are particularly valuable in CI/CD pipelines, cloud environments, and any scenario where network reliability is not guaranteed.
