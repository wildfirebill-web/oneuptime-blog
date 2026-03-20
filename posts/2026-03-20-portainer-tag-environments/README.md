# How to Tag Environments in Portainer for Better Organization

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Environments, Tags, Organization, Edge Computing

Description: Use tags in Portainer to label environments with metadata for filtering, organization, and dynamic edge group creation.

## Introduction

Tags in Portainer are key-value labels attached to environments. Unlike groups (hierarchical), tags are flat and flexible — an environment can have multiple tags. Tags enable filtering in the UI and are the foundation for dynamic edge groups that automatically include environments based on tag matches.

## Creating Tags

Tags must be created globally before assigning to environments.

### Via Web UI

1. Go to **Settings** → **Tags** (or find it in the Environments section)
2. Click **Add tag**
3. Enter the tag name (e.g., "production", "eu-west", "kubernetes")
4. Click **Add**

### Via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Create tags
TAGS=("production" "staging" "development" "us-east" "eu-west" "kubernetes" "docker" "edge")

for tag in "${TAGS[@]}"; do
  RESPONSE=$(curl -s -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    https://portainer.example.com/api/tags \
    -d "{\"name\": \"${tag}\"}")
  TAG_ID=$(echo $RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin).get('ID','error'))")
  echo "Created tag '$tag' with ID: $TAG_ID"
done
```

## Assigning Tags to Environments

```bash
# Get tag IDs
TAGS_LIST=$(curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/tags \
  | python3 -c "import sys,json; [print(f'{t[\"ID\"]}:{t[\"Name\"]}') for t in json.load(sys.stdin)]")

echo "$TAGS_LIST"

# Get environment details
ENDPOINT_ID=1
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/${ENDPOINT_ID}" \
  | python3 -c "import sys,json; e=json.load(sys.stdin); print(f'Name: {e[\"Name\"]}, Tags: {e.get(\"TagIds\",[])}')
"

# Add tags to an environment (replace existing tags)
# Tags 1=production, 2=us-east, 5=kubernetes
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/endpoints/${ENDPOINT_ID}" \
  -d '{
    "TagIds": [1, 2, 5]
  }'
```

## Bulk Tagging Script

```bash
#!/bin/bash
# bulk-tag-environments.sh

TOKEN="your-admin-token"
PORTAINER_URL="https://portainer.example.com"

# Environment to tag: "endpoint_id:tag_ids" (comma-separated)
ASSIGNMENTS=(
  "1:1,2,5"    # Env 1: production, us-east, kubernetes
  "2:1,3,5"    # Env 2: production, eu-west, kubernetes
  "3:4,2,6"    # Env 3: staging, us-east, docker
  "4:7,2"      # Env 4: edge, us-east
)

for assignment in "${ASSIGNMENTS[@]}"; do
  IFS=':' read -r env_id tag_ids <<< "$assignment"

  # Convert comma-separated to JSON array
  TAG_ARRAY=$(echo $tag_ids | python3 -c "import sys; ids=sys.stdin.read().strip().split(','); print('[' + ','.join(ids) + ']')")

  echo "Tagging environment $env_id with tags: $TAG_ARRAY"
  curl -s -X PUT \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "${PORTAINER_URL}/api/endpoints/${env_id}" \
    -d "{\"TagIds\": ${TAG_ARRAY}}"
done
```

## Dynamic Edge Groups Based on Tags

Tags enable dynamic edge groups that automatically include environments matching specified tags:

```bash
# Create a dynamic edge group for all "us-east" environments
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/edge_groups \
  -d '{
    "name": "US-East Sites",
    "dynamic": true,
    "tagIds": [2]
  }'

# Create a dynamic group for all production kubernetes environments
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/edge_groups \
  -d '{
    "name": "Production Kubernetes",
    "dynamic": true,
    "tagIds": [1, 5]
  }'
```

When you tag a new environment with `production` AND `kubernetes`, it automatically joins the "Production Kubernetes" dynamic group.

## Filtering Environments by Tags

In the Portainer UI:
1. Go to the **Environments** page
2. Use the **Filter** input to filter by tag names
3. Environments with matching tags appear

## Tag Naming Conventions

Use consistent naming conventions for tags:

```
# Stage tags
production, staging, development, testing

# Region tags
us-east, us-west, eu-west, eu-central, ap-southeast

# Type tags
kubernetes, docker, swarm, edge, aci

# Team tags
backend-team, frontend-team, platform-team, data-team

# Custom
high-availability, low-latency, gpu-enabled
```

## Conclusion

Tags provide a flexible, multi-dimensional way to describe your environments beyond simple group membership. While groups organize environments hierarchically, tags allow many dimensions simultaneously — a single environment can be "production", "us-east", "kubernetes", and "high-availability" all at once. This multi-dimensional tagging is especially powerful for dynamic edge groups that automatically track environments as tags are assigned or removed.
