# How to Create the Initial Admin User via the Portainer API - User

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Automation, DevOps, Installation

Description: Learn how to create the initial Portainer administrator user via the API during automated deployments, eliminating the need for manual browser-based setup.

## Introduction

When deploying Portainer in automated environments (CI/CD, infrastructure-as-code, scripts), you need to initialize the admin user without browser interaction. Portainer provides an API endpoint specifically for this purpose, allowing you to fully automate the initial setup. This endpoint is only available during the initial setup window before any admin is created.

## Prerequisites

- Portainer CE or BE freshly installed with no admin user created yet
- `curl` and `jq` available on your automation machine
- Network access to the Portainer instance

## Understanding the Initialization Window

After installing Portainer, there is a time window (typically 5 minutes by default) during which an admin user can be created without authentication. After this window closes, you must already be authenticated to create users.

When you access Portainer for the first time, it shows the "Create initial admin user" screen - the API replicates this behavior.

## Step 1: Check If Portainer Needs Initialization

```bash
PORTAINER_URL="https://portainer.example.com"

# Check initialization status

STATUS=$(curl -s "${PORTAINER_URL}/api/system/status")
echo $STATUS | jq .

# If "isAdmin" is false or "status" indicates setup is needed, proceed
IS_INITIALIZED=$(echo $STATUS | jq -r '.isAdmin')
echo "Is admin created: $IS_INITIALIZED"
```

## Step 2: Create the Initial Admin User

```bash
PORTAINER_URL="https://portainer.example.com"
ADMIN_PASSWORD="StrongP@ssw0rd123!"

# Create the initial admin user
RESPONSE=$(curl -s -X POST "${PORTAINER_URL}/api/users/admin/init" \
  -H "Content-Type: application/json" \
  -d "{
    \"username\": \"admin\",
    \"password\": \"${ADMIN_PASSWORD}\"
  }")

echo "Response: $RESPONSE"

# Check for success
if echo "$RESPONSE" | jq -e '.Id' > /dev/null 2>&1; then
  echo "Admin user created successfully!"
  echo "User ID: $(echo $RESPONSE | jq -r '.Id')"
else
  echo "Error creating admin user: $RESPONSE"
fi
```

## Step 3: Verify Admin Was Created

After creation, verify you can authenticate:

```bash
# Try to authenticate with the new admin credentials
TOKEN=$(curl -s -X POST "${PORTAINER_URL}/api/auth" \
  -H "Content-Type: application/json" \
  -d "{
    \"username\": \"admin\",
    \"password\": \"${ADMIN_PASSWORD}\"
  }" | jq -r '.jwt')

if [ "$TOKEN" != "null" ] && [ -n "$TOKEN" ]; then
  echo "Authentication successful! Token obtained."
  echo "Token: ${TOKEN:0:50}..."  # Show first 50 chars
else
  echo "Authentication failed."
fi
```

## Step 4: Full Automation Script

Here is a complete script for automated Portainer initialization:

```bash
#!/bin/bash
# init-portainer.sh - Fully automated Portainer initialization

set -euo pipefail

PORTAINER_URL="${PORTAINER_URL:-https://portainer.example.com}"
ADMIN_USER="${ADMIN_USER:-admin}"
ADMIN_PASS="${ADMIN_PASS:-ChangeMe123!}"
MAX_WAIT_SECONDS=120  # Wait up to 2 minutes for Portainer to start
SLEEP_INTERVAL=5

echo "=== Portainer Initialization Script ==="
echo "URL: $PORTAINER_URL"

# Step 1: Wait for Portainer to be ready
echo "Waiting for Portainer to be ready..."
elapsed=0
until curl -sf "${PORTAINER_URL}/api/system/status" > /dev/null 2>&1; do
  sleep $SLEEP_INTERVAL
  elapsed=$((elapsed + SLEEP_INTERVAL))
  if [ $elapsed -ge $MAX_WAIT_SECONDS ]; then
    echo "Timeout: Portainer did not become ready in ${MAX_WAIT_SECONDS}s"
    exit 1
  fi
  echo "  Still waiting... (${elapsed}s)"
done
echo "Portainer is ready!"

# Step 2: Check if already initialized
STATUS=$(curl -s "${PORTAINER_URL}/api/system/status")
IS_ADMIN=$(echo "$STATUS" | jq -r '.isAdmin // false')

if [ "$IS_ADMIN" = "true" ]; then
  echo "Portainer is already initialized. Skipping admin creation."
  exit 0
fi

# Step 3: Create initial admin user
echo "Creating initial admin user..."
RESPONSE=$(curl -s -X POST "${PORTAINER_URL}/api/users/admin/init" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"${ADMIN_USER}\",\"password\":\"${ADMIN_PASS}\"}")

if echo "$RESPONSE" | jq -e '.Id' > /dev/null 2>&1; then
  echo "Admin user '${ADMIN_USER}' created successfully (ID: $(echo $RESPONSE | jq -r '.Id'))"
else
  echo "ERROR creating admin: $RESPONSE" >&2
  exit 1
fi

# Step 4: Obtain JWT token
echo "Authenticating..."
TOKEN=$(curl -s -X POST "${PORTAINER_URL}/api/auth" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"${ADMIN_USER}\",\"password\":\"${ADMIN_PASS}\"}" | jq -r '.jwt')

echo "Authentication successful."

# Step 5: Additional setup tasks (optional)
# Disable telemetry
curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/settings" \
  -d '{"enableTelemetry": false}' > /dev/null

echo "=== Portainer initialization complete ==="
echo "URL:      $PORTAINER_URL"
echo "Username: $ADMIN_USER"
```

## Step 5: Docker Compose with Auto-Init

Use the script in a Docker Compose setup with a health check:

```yaml
# docker-compose.yml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    ports:
      - "9000:9000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/api/system/status"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  portainer-init:
    image: curlimages/curl:latest
    depends_on:
      portainer:
        condition: service_healthy
    environment:
      PORTAINER_URL: http://portainer:9000
      ADMIN_PASS: ${PORTAINER_ADMIN_PASS:-ChangeMe123!}
    command: >
      sh -c "
        curl -s -X POST $${PORTAINER_URL}/api/users/admin/init
          -H 'Content-Type: application/json'
          -d '{\"username\":\"admin\",\"password\":\"'$${ADMIN_PASS}'\"}' &&
        echo 'Admin initialized'
      "
    restart: "no"

volumes:
  portainer_data:
```

## Conclusion

Creating the initial Portainer admin user via the API is essential for fully automated, reproducible deployments. Use the `/api/users/admin/init` endpoint during the initialization window, combine it with health checks to ensure Portainer is ready, and run it as part of your infrastructure provisioning scripts. Never store the admin password in the script file itself - always use environment variables or a secrets manager.
