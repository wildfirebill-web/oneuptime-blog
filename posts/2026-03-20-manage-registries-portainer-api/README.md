# How to Manage Registries via the Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Container Registry, Automation, DevOps

Description: Learn how to add, update, and manage container registries in Portainer programmatically using the REST API.

## Registry Management Endpoints

| Method | Endpoint | Action |
|--------|----------|--------|
| GET | `/api/registries` | List all registries |
| GET | `/api/registries/{id}` | Get registry details |
| POST | `/api/registries` | Add a new registry |
| PUT | `/api/registries/{id}` | Update a registry |
| DELETE | `/api/registries/{id}` | Remove a registry |

## Listing Registries

```bash
# List all configured registries

curl -s "${PORTAINER_URL}/api/registries" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | {id: .Id, name: .Name, url: .URL, type: .Type}]'
```

Registry types:
- **1** = Custom registry (generic)
- **2** = Docker Hub
- **3** = GitLab
- **5** = AWS ECR
- **6** = Docker Hub (authenticated)
- **7** = GitHub Container Registry (GHCR)

## Adding a Custom Registry

```bash
# Add a private registry with authentication
curl -X POST "${PORTAINER_URL}/api/registries" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Type": 1,
    "Name": "My Private Registry",
    "URL": "registry.mycompany.com",
    "Authentication": true,
    "Username": "myuser",
    "Password": "mypassword"
  }'
```

## Adding Docker Hub (Authenticated)

```bash
# Add Docker Hub with credentials to avoid rate limiting
curl -X POST "${PORTAINER_URL}/api/registries" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Type": 6,
    "Name": "Docker Hub",
    "URL": "docker.io",
    "Authentication": true,
    "Username": "my-dockerhub-username",
    "Password": "dckr_pat_mytoken"
  }'
```

## Adding AWS ECR

```bash
# Add Amazon ECR registry
curl -X POST "${PORTAINER_URL}/api/registries" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Type": 5,
    "Name": "AWS ECR Production",
    "URL": "123456789012.dkr.ecr.us-east-1.amazonaws.com",
    "Authentication": true,
    "Username": "AWS",
    "Password": "'"$(aws ecr get-login-password --region us-east-1)"'",
    "Ecr": {
      "Region": "us-east-1"
    }
  }'
```

## Refreshing ECR Token (Automated)

```bash
#!/bin/bash
# Refresh the ECR token in Portainer every 6 hours

PORTAINER_URL="https://portainer.mycompany.com"
API_TOKEN="${PORTAINER_API_TOKEN}"

# Find the ECR registry ID
ECR_REGISTRY_ID=$(curl -s "${PORTAINER_URL}/api/registries" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '.[] | select(.Type == 5) | .Id')

# Get fresh ECR token
NEW_TOKEN=$(aws ecr get-login-password --region us-east-1)

# Update the registry with the new token
curl -X PUT "${PORTAINER_URL}/api/registries/${ECR_REGISTRY_ID}" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"Type\": 5,
    \"Name\": \"AWS ECR Production\",
    \"URL\": \"123456789012.dkr.ecr.us-east-1.amazonaws.com\",
    \"Authentication\": true,
    \"Username\": \"AWS\",
    \"Password\": \"${NEW_TOKEN}\"
  }"

echo "ECR token refreshed at $(date)"
```

## Assigning a Registry to an Environment

```bash
# Assign a registry to a specific environment
curl -X PUT "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/registries/${REGISTRY_ID}" \
  -H "Authorization: Bearer ${API_TOKEN}"
```

## Deleting a Registry

```bash
# Remove a registry from Portainer
curl -X DELETE "${PORTAINER_URL}/api/registries/${REGISTRY_ID}" \
  -H "Authorization: Bearer ${API_TOKEN}"
```

## Conclusion

The Portainer registries API enables centralized registry management for multi-environment setups. Use it especially to automate ECR token rotation, which is essential since ECR tokens expire every 12 hours.
