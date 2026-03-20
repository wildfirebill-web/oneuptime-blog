# How to Manage Registries via the Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Registry, Docker, Automation

Description: Learn how to add, configure, and manage container image registries in Portainer via the REST API to enable private image deployments across your environments.

## Introduction

Container image registries store and serve container images. Portainer supports connecting to multiple private and public registries, storing credentials securely so containers can pull images without exposing credentials to end users. The Portainer API lets you automate registry management for consistent multi-environment setups.

## Prerequisites

- Portainer CE or BE with admin access
- Valid admin JWT token or API access token
- Registry credentials (URL, username, password)

## Supported Registry Types

| Type | Value | Examples |
|------|-------|---------|
| Custom (generic) | 6 | Any registry with Docker Registry API |
| DockerHub | 1 | hub.docker.com |
| Quay.io | 3 | quay.io |
| Azure ACR | 4 | yourregistry.azurecr.io |
| Custom GitLab | 5 | registry.gitlab.com |
| GitHub CR | 7 | ghcr.io |
| ProGet | 8 | proget.yourcompany.com |

## Step 1: List All Configured Registries

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"

# List all registries
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/registries" | \
  jq '.[] | {id: .Id, name: .Name, type: .Type, url: .URL}'
```

## Step 2: Add a Private Registry

```bash
# Add a generic private registry (e.g., self-hosted Harbor or generic registry)
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/registries" \
  -d '{
    "Name": "Company Harbor",
    "Type": 6,
    "URL": "registry.company.com",
    "Authentication": true,
    "Username": "portainer-svc",
    "Password": "registrypassword"
  }' | jq .
```

## Step 3: Add Docker Hub (Private)

```bash
# Add Docker Hub with credentials (for private repos and rate limit bypass)
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/registries" \
  -d '{
    "Name": "Docker Hub",
    "Type": 1,
    "URL": "https://index.docker.io",
    "Authentication": true,
    "Username": "your-dockerhub-username",
    "Password": "your-dockerhub-token"
  }' | jq .
```

## Step 4: Add GitHub Container Registry (GHCR)

```bash
# Add GitHub Container Registry
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/registries" \
  -d '{
    "Name": "GitHub Container Registry",
    "Type": 7,
    "URL": "ghcr.io",
    "Authentication": true,
    "Username": "your-github-username",
    "Password": "ghp_your_github_personal_access_token"
  }' | jq .
```

## Step 5: Add Azure Container Registry (ACR)

```bash
# Add Azure Container Registry
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/registries" \
  -d '{
    "Name": "Azure Production ACR",
    "Type": 4,
    "URL": "yourregistry.azurecr.io",
    "Authentication": true,
    "Username": "service-principal-client-id",
    "Password": "service-principal-secret"
  }' | jq .
```

## Step 6: Inspect a Registry

```bash
REGISTRY_ID=2

# Get registry details
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/registries/${REGISTRY_ID}" | jq .

# Note: Password is never returned in API responses
```

## Step 7: Update a Registry

```bash
REGISTRY_ID=2

# Update registry credentials
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/registries/${REGISTRY_ID}" \
  -d '{
    "Name": "Company Harbor (Updated)",
    "URL": "registry.company.com",
    "Authentication": true,
    "Username": "portainer-svc",
    "Password": "new-registry-password"
  }' | jq .
```

## Step 8: Delete a Registry

```bash
REGISTRY_ID=2

# Delete registry configuration from Portainer
curl -s -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/registries/${REGISTRY_ID}"

echo "Registry $REGISTRY_ID removed."
```

## Step 9: List Registry Repositories and Tags

```bash
REGISTRY_ID=2

# List all repositories in a registry
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/registries/${REGISTRY_ID}/repositories" | jq .

# List tags for a specific repository
REPO_NAME="myapp"
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/registries/${REGISTRY_ID}/repositories/${REPO_NAME}/tags" | jq .
```

## Step 10: Automated Registry Setup Script

```bash
#!/bin/bash
# setup-registries.sh — Configure all registries from environment variables

PORTAINER_URL="https://portainer.example.com"
TOKEN=$(curl -s -X POST "${PORTAINER_URL}/api/auth" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' | jq -r '.jwt')

# Registry configurations
declare -A REGISTRIES
REGISTRIES["Company Harbor"]="6|registry.company.com|harbor-svc|${HARBOR_PASSWORD}"
REGISTRIES["Docker Hub"]="1|https://index.docker.io|${DOCKERHUB_USER}|${DOCKERHUB_TOKEN}"
REGISTRIES["GitHub CR"]="7|ghcr.io|${GITHUB_USER}|${GITHUB_TOKEN}"

for REG_NAME in "${!REGISTRIES[@]}"; do
  IFS='|' read -r TYPE URL USERNAME PASSWORD <<< "${REGISTRIES[$REG_NAME]}"

  echo "Adding registry: $REG_NAME..."

  RESPONSE=$(curl -s -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "${PORTAINER_URL}/api/registries" \
    -d "{
      \"Name\": \"$REG_NAME\",
      \"Type\": $TYPE,
      \"URL\": \"$URL\",
      \"Authentication\": true,
      \"Username\": \"$USERNAME\",
      \"Password\": \"$PASSWORD\"
    }")

  ID=$(echo $RESPONSE | jq -r '.Id // empty')
  if [ -n "$ID" ]; then
    echo "  Added '$REG_NAME' (ID: $ID)"
  else
    echo "  ERROR: $RESPONSE"
  fi
done

echo "Registry setup complete."
```

## Conclusion

Managing container registries via the Portainer API enables consistent, automated credential management across multiple Portainer instances. Configure registries as part of your infrastructure provisioning pipeline, rotate credentials programmatically when they expire, and ensure all environments have access to the same registries. Always use service accounts or access tokens for registry credentials rather than personal user credentials.
