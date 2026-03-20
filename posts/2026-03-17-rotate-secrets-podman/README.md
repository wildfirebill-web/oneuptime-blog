# How to Rotate Secrets in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Secrets, Rotation, Security

Description: Learn how to rotate secrets in Podman to maintain security by regularly updating credentials and sensitive data.

---

> Regular secret rotation limits the window of exposure if a secret is compromised and is a fundamental security practice for production environments.

Secret rotation involves replacing existing secrets with new values and ensuring all containers pick up the updated credentials. Podman does not have a built-in rotation command, but the process is straightforward using remove-and-recreate combined with container restarts.

---

## Basic Secret Rotation

```bash
# Step 1: Remove the old secret (stop containers first if needed)

podman stop my-app
podman rm my-app

podman secret rm db_password

# Step 2: Create the new secret with the updated value
echo -n "new-rotated-password-2026" | podman secret create db_password -

# Step 3: Restart the container with the updated secret
podman run -d \
  --name my-app \
  --secret db_password \
  my-app:latest
```

## Rotation Script

```bash
#!/bin/bash
# rotate-secret.sh - Rotate a Podman secret with minimal downtime

SECRET_NAME="$1"
NEW_VALUE="$2"
CONTAINERS="$3"  # Comma-separated container names

if [ -z "$SECRET_NAME" ] || [ -z "$NEW_VALUE" ]; then
  echo "Usage: rotate-secret.sh <secret-name> <new-value> [container1,container2]"
  exit 1
fi

echo "Rotating secret: $SECRET_NAME"

# Stop containers using the secret
if [ -n "$CONTAINERS" ]; then
  IFS=',' read -ra CONTAINER_LIST <<< "$CONTAINERS"
  for container in "${CONTAINER_LIST[@]}"; do
    echo "Stopping container: $container"
    podman stop "$container" 2>/dev/null
  done
fi

# Remove and recreate the secret
podman secret rm "$SECRET_NAME" 2>/dev/null
printf '%s' "$NEW_VALUE" | podman secret create "$SECRET_NAME" -
echo "Secret $SECRET_NAME rotated successfully"

# Restart containers
if [ -n "$CONTAINERS" ]; then
  for container in "${CONTAINER_LIST[@]}"; do
    echo "Starting container: $container"
    podman start "$container"
  done
fi
```

## Zero-Downtime Rotation with Blue-Green

```bash
#!/bin/bash
# Zero-downtime rotation using blue-green deployment

# Step 1: Create the new secret with a temporary name
echo -n "new-password-value" | podman secret create db_password_new -

# Step 2: Start a new container with the new secret
podman run -d \
  --name my-app-new \
  --secret db_password_new \
  -p 8081:8080 \
  my-app:latest

# Step 3: Verify the new container is healthy
sleep 10
if podman healthcheck run my-app-new; then
  echo "New container is healthy"

  # Step 4: Switch traffic (update load balancer, proxy, etc.)
  # ... your traffic switching logic here ...

  # Step 5: Stop the old container and clean up
  podman stop my-app
  podman rm my-app
  podman secret rm db_password

  # Step 6: Rename for consistency
  echo -n "new-password-value" | podman secret create db_password -
  podman secret rm db_password_new
else
  echo "New container is unhealthy, rolling back"
  podman stop my-app-new
  podman rm my-app-new
  podman secret rm db_password_new
fi
```

## Automated Rotation Schedule

```bash
#!/bin/bash
# scheduled-rotation.sh - Run via cron for periodic rotation

TIMESTAMP=$(date +%s)
NEW_PASSWORD=$(openssl rand -base64 32)

# Rotate the database password
podman stop my-app
podman secret rm db_password
echo -n "$NEW_PASSWORD" | podman secret create db_password -

# Update the database password as well
podman exec my-db psql -U postgres -c "ALTER USER app PASSWORD '$NEW_PASSWORD';"

# Restart the application
podman start my-app

echo "Secret rotated at $(date)"
```

## Summary

Secret rotation in Podman involves removing the old secret, creating a new one with updated values, and restarting containers. For production environments, use blue-green deployments to achieve zero-downtime rotation. Automate the process with scripts and run them on a schedule using cron or a similar scheduler. Regular rotation limits the impact of compromised credentials and is an essential security practice.
