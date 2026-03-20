# How to Organize Environments with Groups in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Environments, Groups, Organization, Management

Description: Create and use environment groups in Portainer to logically organize multiple environments for better navigation and bulk access control management.

## Introduction

As your Portainer installation grows to manage multiple environments, the environment list can become unwieldy. Environment Groups let you organize environments into logical categories — by stage (dev/staging/prod), region, team, or project. This guide covers creating and managing environment groups.

## Creating Groups via Web UI

1. Go to **Environments** in the left sidebar
2. Click on **Groups** tab at the top
3. Click **Add group**
4. Fill in:
   - **Name**: e.g., "Production", "Europe", "Team A"
   - **Description**: Optional description
   - **Environments**: Select which environments to include
5. Click **Create group**

## Creating Groups via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Create a group with environments
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoint_groups \
  -d '{
    "name": "Production",
    "description": "All production environments",
    "associatedEndpoints": [1, 2, 3]
  }'

# Get all groups
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/endpoint_groups \
  | python3 -m json.tool

# Update a group
GROUP_ID=1
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/endpoint_groups/${GROUP_ID}" \
  -d '{
    "name": "Production",
    "description": "Production environments - all regions",
    "associatedEndpoints": [1, 2, 3, 4]
  }'

# Delete a group
curl -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoint_groups/${GROUP_ID}"
```

## Adding Environments to a Group

```bash
# Add a single environment to a group
ENDPOINT_ID=5
GROUP_ID=1

curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoint_groups/${GROUP_ID}/endpoints/${ENDPOINT_ID}"

# Remove an environment from a group
curl -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoint_groups/${GROUP_ID}/endpoints/${ENDPOINT_ID}"
```

## Recommended Group Structures

### By Stage

```
Groups:
├── Production (envs: prod-us, prod-eu, prod-ap)
├── Staging (envs: staging-us, staging-eu)
└── Development (envs: dev-all, local)
```

### By Region

```
Groups:
├── US (envs: us-east-prod, us-east-staging, us-west-prod)
├── Europe (envs: eu-west-prod, eu-central-staging)
└── Asia-Pacific (envs: ap-southeast-prod)
```

### By Team

```
Groups:
├── Platform Team (all environments)
├── Backend Team (backend-dev, backend-staging, backend-prod)
├── Frontend Team (frontend-dev, frontend-staging)
└── Data Team (data-processing-prod)
```

## Bulk Operations on Groups

Groups enable bulk access assignments (covered in the per-group access control guide), but also serve as filters in the UI:

```bash
# List all environments in a group
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoint_groups/${GROUP_ID}" \
  | python3 -c "
import sys, json
group = json.load(sys.stdin)
print(f'Group: {group[\"Name\"]}')
print(f'Environments: {group.get(\"AssociatedEndpoints\", [])}')
"

# Apply tags to all environments in a group via script
GROUP_ENVS=$(curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoint_groups/${GROUP_ID}" \
  | python3 -c "import sys,json; print(' '.join(map(str, json.load(sys.stdin).get('AssociatedEndpoints', []))))")

for env_id in $GROUP_ENVS; do
  echo "Processing environment $env_id"
  # Apply tags, access policies, etc.
done
```

## Conclusion

Environment groups are a simple but powerful organizational feature in Portainer. They don't just provide visual organization — they also serve as the basis for bulk access control assignments. Invest time in defining a good group structure early, as it determines how intuitively your team navigates a large Portainer installation. The most common and useful structure is organizing by deployment stage (dev/staging/prod).
