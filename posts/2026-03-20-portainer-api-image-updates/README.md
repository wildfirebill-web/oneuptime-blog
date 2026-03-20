# How to Automate Image Updates via Portainer API - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Docker, Images, Automation

Description: Automate Docker image updates across your Portainer-managed environments using the Portainer REST API and webhooks.

## Introduction

Keeping Docker images up-to-date is essential for security and feature delivery. Portainer's API allows you to pull new images and redeploy containers or stacks programmatically, enabling automated update pipelines triggered by CI/CD, schedule, or registry webhooks.

## Prerequisites

- Portainer CE or BE with API access
- Docker registry (Docker Hub, GHCR, private registry)
- Python or shell scripting environment
- Registry webhook support (optional)

## Method 1: Pull and Redeploy via API

```bash
#!/bin/bash
# update-container.sh

# Usage: ./update-container.sh <container-name> <new-image-tag>

PORTAINER_URL="https://portainer.example.com"
API_KEY="your-api-key"
ENDPOINT_ID=1
CONTAINER_NAME="${1:-my-app}"
NEW_IMAGE="${2:-myapp:latest}"

# Helper function for API calls
portainer_api() {
    curl -s \
        -H "X-API-Key: $API_KEY" \
        -H "Content-Type: application/json" \
        "$@"
}

echo "=== Portainer Image Update ==="
echo "Container: $CONTAINER_NAME"
echo "New Image: $NEW_IMAGE"

# Step 1: Pull the new image
echo "[1/4] Pulling new image: $NEW_IMAGE"
portainer_api -X POST \
    "$PORTAINER_URL/api/endpoints/$ENDPOINT_ID/docker/images/create?fromImage=$NEW_IMAGE"

# Step 2: Find the container
echo "[2/4] Finding container: $CONTAINER_NAME"
CONTAINER_ID=$(portainer_api \
    "$PORTAINER_URL/api/endpoints/$ENDPOINT_ID/docker/containers/json?all=true" \
    | python3 -c "
import sys, json
containers = json.load(sys.stdin)
for c in containers:
    if any('$CONTAINER_NAME' in n for n in c['Names']):
        print(c['Id'])
        break
")

if [ -z "$CONTAINER_ID" ]; then
    echo "Error: Container $CONTAINER_NAME not found"
    exit 1
fi

echo "Found container ID: ${CONTAINER_ID:0:12}"

# Step 3: Get container configuration
echo "[3/4] Getting container configuration..."
CONTAINER_CONFIG=$(portainer_api \
    "$PORTAINER_URL/api/endpoints/$ENDPOINT_ID/docker/containers/$CONTAINER_ID/json")

# Extract config for recreation
HOST_CONFIG=$(echo $CONTAINER_CONFIG | python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps(d['HostConfig']))")
ENV=$(echo $CONTAINER_CONFIG | python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps(d['Config']['Env']))")

# Step 4: Stop, remove, and recreate with new image
echo "[4/4] Replacing container..."

# Stop container
portainer_api -X POST \
    "$PORTAINER_URL/api/endpoints/$ENDPOINT_ID/docker/containers/$CONTAINER_ID/stop"

# Remove container
portainer_api -X DELETE \
    "$PORTAINER_URL/api/endpoints/$ENDPOINT_ID/docker/containers/$CONTAINER_ID"

# Create new container with updated image
NEW_CONTAINER=$(portainer_api -X POST \
    "$PORTAINER_URL/api/endpoints/$ENDPOINT_ID/docker/containers/create?name=$CONTAINER_NAME" \
    -d "{
        \"Image\": \"$NEW_IMAGE\",
        \"Env\": $ENV,
        \"HostConfig\": $HOST_CONFIG
    }")

NEW_ID=$(echo $NEW_CONTAINER | python3 -c "import sys,json; print(json.load(sys.stdin)['Id'])")

# Start new container
portainer_api -X POST \
    "$PORTAINER_URL/api/endpoints/$ENDPOINT_ID/docker/containers/$NEW_ID/start"

echo "Container updated successfully!"
echo "New container ID: ${NEW_ID:0:12}"
```

## Method 2: Update a Stack with New Image Tags

```python
#!/usr/bin/env python3
# update_stack_image.py

import requests
import re
import sys

PORTAINER_URL = "https://portainer.example.com"
API_KEY = "your-api-key"
ENDPOINT_ID = 1

headers = {
    "X-API-Key": API_KEY,
    "Content-Type": "application/json"
}


def get_stack_by_name(stack_name: str) -> dict:
    """Find a stack by name."""
    resp = requests.get(
        f"{PORTAINER_URL}/api/stacks?endpointId={ENDPOINT_ID}",
        headers=headers
    )
    resp.raise_for_status()
    stacks = resp.json()
    for stack in stacks:
        if stack['Name'] == stack_name:
            return stack
    return None


def get_stack_file(stack_id: int) -> str:
    """Get the compose file content for a stack."""
    resp = requests.get(
        f"{PORTAINER_URL}/api/stacks/{stack_id}/file",
        headers=headers
    )
    resp.raise_for_status()
    return resp.json()['StackFileContent']


def update_image_in_compose(compose_content: str, service: str, new_tag: str) -> str:
    """Update the image tag for a service in compose content."""
    # Match: "image: registry/name:old-tag" or "image: name:old-tag"
    pattern = rf'([ \t]*image:\s+[^\s:]+):[^\s\n]+'

    lines = compose_content.split('\n')
    in_service = False
    result = []

    for line in lines:
        if re.match(rf'^\s{2}{service}:', line):
            in_service = True
        elif re.match(r'^\s{2}\w+:', line) and not line.startswith('      '):
            in_service = False

        if in_service and re.match(r'\s+image:', line):
            # Extract image name without tag
            parts = line.split('image:')
            image_part = parts[1].strip()
            image_name = image_part.split(':')[0]
            line = f"{parts[0]}image: {image_name}:{new_tag}"

        result.append(line)

    return '\n'.join(result)


def update_stack(stack_id: int, new_content: str, env_vars: list = None):
    """Update a stack with new compose content."""
    resp = requests.put(
        f"{PORTAINER_URL}/api/stacks/{stack_id}?endpointId={ENDPOINT_ID}",
        headers=headers,
        json={
            "StackFileContent": new_content,
            "Env": env_vars or [],
            "Prune": False
        }
    )
    resp.raise_for_status()
    return resp.json()


def main():
    stack_name = sys.argv[1] if len(sys.argv) > 1 else "my-app"
    service_name = sys.argv[2] if len(sys.argv) > 2 else "api"
    new_tag = sys.argv[3] if len(sys.argv) > 3 else "latest"

    print(f"Updating {stack_name}/{service_name} to tag: {new_tag}")

    # Get the stack
    stack = get_stack_by_name(stack_name)
    if not stack:
        print(f"Stack '{stack_name}' not found")
        sys.exit(1)

    # Get the compose file
    compose_content = get_stack_file(stack['Id'])

    # Update the image tag
    new_compose = update_image_in_compose(compose_content, service_name, new_tag)

    # Update the stack
    result = update_stack(stack['Id'], new_compose)
    print(f"Stack updated successfully! Status: {result.get('Status')}")


if __name__ == '__main__':
    main()
```

## Method 3: Using Portainer Webhooks for Auto-Updates

Portainer supports webhooks that trigger stack redeployments:

```bash
# In Portainer UI: Stacks > Your Stack > Webhooks > Enable
# Copy the webhook URL

# Trigger a stack update (pulls latest images and redeploys)
curl -X POST "https://portainer.example.com/api/webhooks/your-webhook-token"

# Integrate with Docker Hub webhooks
# In Docker Hub: Repository > Webhooks > Add webhook URL
```

## CI/CD Integration

```yaml
# GitHub Actions
name: Deploy to Production
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Update image in Portainer
        run: |
          python update_stack_image.py my-app api ${{ github.sha }}
        env:
          PORTAINER_URL: ${{ secrets.PORTAINER_URL }}
          PORTAINER_API_KEY: ${{ secrets.PORTAINER_API_KEY }}
```

## Conclusion

Automating image updates via the Portainer API keeps your containers running the latest, most secure images without manual intervention. Whether triggered by CI/CD pipelines, registry webhooks, or scheduled jobs, automated image updates reduce the operational burden of container management and improve your security posture.
