# How to Automate Portainer Configuration with API Scripts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Automation, Infrastructure as Code, DevOps

Description: Learn how to write comprehensive automation scripts to configure Portainer programmatically for repeatable, version-controlled setups.

## Why Automate Portainer Configuration?

Manually configuring Portainer through the UI is not repeatable or version-controlled. Automating it with API scripts enables:

- **Idempotent setups**: Run the same script multiple times safely.
- **Disaster recovery**: Re-create Portainer configuration from scratch.
- **Environment parity**: Keep development and production configs in sync.
- **CI/CD integration**: Bootstrap new environments automatically.

## Configuration Script Structure

```bash
#!/bin/bash
# portainer-configure.sh - Complete Portainer setup automation
set -euo pipefail

PORTAINER_URL="${PORTAINER_URL:?Required}"
ADMIN_USER="${PORTAINER_ADMIN_USER:-admin}"
ADMIN_PASS="${PORTAINER_ADMIN_PASS:?Required}"

log() { echo "[$(date '+%H:%M:%S')] $*"; }

# 1. Wait for Portainer to be ready
wait_for_portainer() {
  log "Waiting for Portainer..."
  until curl -sf "${PORTAINER_URL}/api/system/status" > /dev/null 2>&1; do
    sleep 3
  done
  log "Portainer is ready"
}

# 2. Initialize or authenticate
get_token() {
  # Try to initialize first (only works once)
  TOKEN=$(curl -sf -X POST "${PORTAINER_URL}/api/users/admin/init" \
    -H "Content-Type: application/json" \
    -d "{\"Username\":\"${ADMIN_USER}\",\"Password\":\"${ADMIN_PASS}\"}" \
    2>/dev/null | jq -r '.jwt' || true)

  # Fall back to login if already initialized
  if [ -z "${TOKEN}" ] || [ "${TOKEN}" = "null" ]; then
    TOKEN=$(curl -sf -X POST "${PORTAINER_URL}/api/auth" \
      -H "Content-Type: application/json" \
      -d "{\"username\":\"${ADMIN_USER}\",\"password\":\"${ADMIN_PASS}\"}" \
      | jq -r '.jwt')
  fi

  echo "${TOKEN}"
}

# 3. Configure global settings
configure_settings() {
  local token="$1"
  log "Configuring settings..."

  curl -sf -X PUT "${PORTAINER_URL}/api/settings" \
    -H "Authorization: Bearer ${token}" \
    -H "Content-Type: application/json" \
    -d '{
      "enableTelemetry": false,
      "authenticationMethod": 1,
      "edgeAgentCheckinInterval": 5,
      "loginBannerEnabled": false
    }'
}

# 4. Add registries
add_registry() {
  local token="$1" name="$2" url="$3" user="$4" pass="$5"
  log "Adding registry: ${name}"

  curl -sf -X POST "${PORTAINER_URL}/api/registries" \
    -H "Authorization: Bearer ${token}" \
    -H "Content-Type: application/json" \
    -d "{
      \"Type\": 1,
      \"Name\": \"${name}\",
      \"URL\": \"${url}\",
      \"Authentication\": true,
      \"Username\": \"${user}\",
      \"Password\": \"${pass}\"
    }"
}

# 5. Create teams
create_team() {
  local token="$1" name="$2"
  log "Creating team: ${name}"

  curl -sf -X POST "${PORTAINER_URL}/api/teams" \
    -H "Authorization: Bearer ${token}" \
    -H "Content-Type: application/json" \
    -d "{\"Name\": \"${name}\"}" | jq '.Id'
}

# Main execution
wait_for_portainer
TOKEN=$(get_token)
configure_settings "${TOKEN}"
add_registry "${TOKEN}" "Harbor" "harbor.mycompany.com" "robot\$portainer" "${HARBOR_TOKEN}"
BACKEND_TEAM=$(create_team "${TOKEN}" "backend")
FRONTEND_TEAM=$(create_team "${TOKEN}" "frontend")

log "Portainer configuration complete!"
```

## Environment Variables for Configuration

```bash
# Store all configuration in environment variables or a .env file
export PORTAINER_URL="https://portainer.mycompany.com"
export PORTAINER_ADMIN_USER="admin"
export PORTAINER_ADMIN_PASS="$(vault kv get -field=password kv/portainer/admin)"
export HARBOR_TOKEN="$(vault kv get -field=token kv/portainer/harbor)"

# Run the configuration script
./portainer-configure.sh
```

## Storing Configuration in Git

```
portainer-config/
├── portainer-configure.sh     # Main setup script
├── settings.json              # Global settings template
├── registries/
│   ├── harbor.json
│   └── ghcr.json
├── teams.txt                  # Team names (one per line)
└── environments/
    ├── production.json
    └── staging.json
```

## Conclusion

API-based Portainer configuration automation brings infrastructure-as-code principles to your container management platform. Store these scripts in Git alongside your application code for a fully version-controlled, reproducible setup.
