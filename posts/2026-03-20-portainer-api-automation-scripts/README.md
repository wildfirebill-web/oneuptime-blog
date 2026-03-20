# How to Automate Portainer Configuration with API Scripts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Automation, DevOps, Infrastructure

Description: Learn how to write comprehensive automation scripts using the Portainer API to configure environments, users, registries, and stacks in a repeatable, infrastructure-as-code style.

## Introduction

The Portainer API enables you to script your entire Portainer configuration — from initial setup through user provisioning, registry configuration, and stack deployments. This guide shows how to build idempotent automation scripts that can provision a fresh Portainer instance or update an existing one consistently.

## Prerequisites

- Portainer CE or BE instance (fresh or existing)
- Bash, Python, or similar scripting environment
- `curl`, `jq` installed
- Access to your container image registry credentials

## Script Design Principles

1. **Idempotent**: Running the script multiple times produces the same result
2. **Check before create**: Avoid duplicates by checking if resources exist first
3. **Use environment variables**: Never hardcode credentials in scripts
4. **Log clearly**: Output progress and errors with timestamps
5. **Exit on error**: Use `set -euo pipefail` in bash scripts

## Complete Portainer Bootstrap Script

```bash
#!/bin/bash
# portainer-bootstrap.sh — Idempotent Portainer configuration

set -euo pipefail

# ===== Configuration (set via environment variables) =====
PORTAINER_URL="${PORTAINER_URL:-https://portainer.example.com}"
ADMIN_USER="${ADMIN_USER:-admin}"
ADMIN_PASS="${ADMIN_PASS:?ADMIN_PASS is required}"
DOCKER_HOST_URL="${DOCKER_HOST_URL:-unix:///var/run/docker.sock}"
REGISTRY_URL="${REGISTRY_URL:-registry.company.com}"
REGISTRY_USER="${REGISTRY_USER:-portainer-svc}"
REGISTRY_PASS="${REGISTRY_PASS:?REGISTRY_PASS is required}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

# ===== Helper Functions =====

api_get() {
  curl -s -H "Authorization: Bearer $TOKEN" "$PORTAINER_URL/api/$1"
}

api_post() {
  curl -s -X POST -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "$PORTAINER_URL/api/$1" -d "$2"
}

api_put() {
  curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "$PORTAINER_URL/api/$1" -d "$2"
}

# ===== Step 1: Wait for Portainer =====
log "Waiting for Portainer to start..."
until curl -sf "$PORTAINER_URL/api/system/status" > /dev/null 2>&1; do
  sleep 3
done
log "Portainer is ready."

# ===== Step 2: Initialize or authenticate =====
IS_INITIALIZED=$(curl -s "$PORTAINER_URL/api/system/status" | jq -r '.isAdmin')

if [ "$IS_INITIALIZED" = "false" ]; then
  log "Initializing admin user..."
  curl -s -X POST "$PORTAINER_URL/api/users/admin/init" \
    -H "Content-Type: application/json" \
    -d "{\"username\":\"$ADMIN_USER\",\"password\":\"$ADMIN_PASS\"}" > /dev/null
  log "Admin user created."
fi

TOKEN=$(curl -s -X POST "$PORTAINER_URL/api/auth" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"$ADMIN_USER\",\"password\":\"$ADMIN_PASS\"}" | jq -r '.jwt')
log "Authenticated."

# ===== Step 3: Configure settings =====
log "Configuring global settings..."
api_put "settings" '{
  "enableTelemetry": false,
  "authenticationMethod": 1,
  "snapshotInterval": "5m"
}' > /dev/null
log "Settings updated."

# ===== Step 4: Add environment (idempotent) =====
EXISTING_EP=$(api_get "endpoints" | jq -r '.[] | select(.Name == "production") | .Id // empty')

if [ -z "$EXISTING_EP" ]; then
  log "Creating 'production' environment..."
  EP=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" \
    -F "Name=production" \
    -F "EndpointCreationType=1" \
    -F "URL=$DOCKER_HOST_URL" \
    "$PORTAINER_URL/api/endpoints")
  EP_ID=$(echo $EP | jq -r '.Id')
  log "Environment 'production' created (ID: $EP_ID)."
else
  EP_ID=$EXISTING_EP
  log "Environment 'production' already exists (ID: $EP_ID)."
fi

# ===== Step 5: Add registry (idempotent) =====
EXISTING_REG=$(api_get "registries" | jq -r --arg url "$REGISTRY_URL" '.[] | select(.URL == $url) | .Id // empty')

if [ -z "$EXISTING_REG" ]; then
  log "Adding registry '$REGISTRY_URL'..."
  api_post "registries" "{
    \"Name\": \"Company Registry\",
    \"Type\": 6,
    \"URL\": \"$REGISTRY_URL\",
    \"Authentication\": true,
    \"Username\": \"$REGISTRY_USER\",
    \"Password\": \"$REGISTRY_PASS\"
  }" > /dev/null
  log "Registry added."
else
  log "Registry '$REGISTRY_URL' already exists."
fi

# ===== Step 6: Create teams (idempotent) =====
for TEAM in "devops" "backend" "frontend"; do
  EXISTING=$(api_get "teams" | jq -r --arg n "$TEAM" '.[] | select(.Name == $n) | .Id // empty')
  if [ -z "$EXISTING" ]; then
    api_post "teams" "{\"name\": \"$TEAM\"}" > /dev/null
    log "Team '$TEAM' created."
  else
    log "Team '$TEAM' already exists."
  fi
done

# ===== Step 7: Deploy a default stack (idempotent) =====
STACK_NAME="monitoring"
EXISTING_STACK=$(api_get "stacks" | jq -r --arg n "$STACK_NAME" '.[] | select(.Name == $n) | .Id // empty')

COMPOSE_CONTENT=$(cat << 'EOF'
version: "3.8"
services:
  portainer-agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
EOF
)

if [ -z "$EXISTING_STACK" ]; then
  log "Creating stack '$STACK_NAME'..."
  api_post "stacks/create/standalone/string?endpointId=${EP_ID}" "{
    \"name\": \"$STACK_NAME\",
    \"stackFileContent\": $(echo "$COMPOSE_CONTENT" | jq -Rs .)
  }" > /dev/null
  log "Stack '$STACK_NAME' deployed."
else
  log "Stack '$STACK_NAME' already exists."
fi

log "=== Portainer bootstrap complete ==="
```

## Running the Script

```bash
# Set environment variables and run
export PORTAINER_URL="https://portainer.example.com"
export ADMIN_PASS="SecureAdminPass123!"
export REGISTRY_URL="registry.company.com"
export REGISTRY_PASS="registry-service-password"

chmod +x portainer-bootstrap.sh
./portainer-bootstrap.sh
```

## Integrating with Terraform or Ansible

```bash
# Use from Terraform null_resource
# null_resource.tf
resource "null_resource" "portainer_bootstrap" {
  provisioner "local-exec" {
    command = "./portainer-bootstrap.sh"
    environment = {
      PORTAINER_URL = var.portainer_url
      ADMIN_PASS    = var.portainer_admin_pass
      REGISTRY_PASS = var.registry_password
    }
  }
  depends_on = [docker_container.portainer]
}
```

## Conclusion

API-driven Portainer configuration enables reproducible, version-controlled infrastructure setup. Write idempotent scripts that check before creating, use environment variables for all secrets, and integrate with your IaC toolchain. These scripts can be committed to your infrastructure repository and run as part of your environment provisioning pipeline, ensuring Portainer is always configured consistently across all deployments.
