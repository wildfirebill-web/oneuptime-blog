# How to Trigger Webhooks via the Portainer API - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Webhooks, CI/CD, Automation

Description: Learn how to create, list, and trigger Portainer webhooks programmatically via the REST API for automated deployments.

## What Are Portainer Webhooks?

Portainer webhooks are unique URLs that trigger a redeploy of a stack or service when called via HTTP POST. They allow CI/CD systems to trigger deployments without needing full Portainer API credentials.

## Listing Webhooks

```bash
# List all webhooks

curl -s "${PORTAINER_URL}/api/webhooks" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | {id: .Id, token: .Token, resourceId: .ResourceID, type: .Type}]'
```

## Creating a Webhook via API

```bash
# Create a webhook for a stack (Type 1 = stack webhook)
curl -X POST "${PORTAINER_URL}/api/webhooks" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "ResourceID": "my-stack",
    "EndpointID": 1,
    "WebhookType": 1
  }'

# Response includes the token
# {"Id": "...", "Token": "abc123-token-here", ...}
```

## Triggering a Webhook

```bash
# Trigger a webhook to redeploy a stack
WEBHOOK_TOKEN="abc123-token-here"

curl -X POST "${PORTAINER_URL}/api/webhooks/${WEBHOOK_TOKEN}"
# Returns 204 No Content on success

# Trigger with a specific image tag
curl -X POST "${PORTAINER_URL}/api/webhooks/${WEBHOOK_TOKEN}?tag=v2.1.0"
```

## Webhook Types

| Type | Value | Target |
|------|-------|--------|
| Stack | 1 | Re-deploys a Docker Compose stack |
| Service | 2 | Updates a Swarm service image |
| Container | 3 | Recreates a container with latest image |

## Automating Webhook Triggers in CI/CD

```bash
#!/bin/bash
# Trigger a Portainer redeploy after pushing a new Docker image

IMAGE_TAG="${1:-latest}"
PORTAINER_WEBHOOK_URL="${PORTAINER_WEBHOOK_URL:?Required}"

echo "Triggering deployment of tag: ${IMAGE_TAG}"

RESPONSE=$(curl -s -w "\n%{http_code}" -X POST \
  "${PORTAINER_WEBHOOK_URL}?tag=${IMAGE_TAG}")

HTTP_CODE=$(echo "$RESPONSE" | tail -1)

if [ "$HTTP_CODE" -eq 204 ]; then
  echo "Deployment triggered successfully"
else
  echo "Deployment trigger failed with HTTP ${HTTP_CODE}"
  exit 1
fi
```

## GitHub Actions Integration

```yaml
# .github/workflows/deploy.yml
- name: Deploy to production
  env:
    PORTAINER_WEBHOOK_URL: ${{ secrets.PORTAINER_WEBHOOK_URL }}
  run: |
    HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
      -X POST "${PORTAINER_WEBHOOK_URL}?tag=${{ github.sha }}")

    if [ "$HTTP_STATUS" != "204" ]; then
      echo "Deployment failed: HTTP $HTTP_STATUS"
      exit 1
    fi
    echo "Deployed successfully"
```

## Deleting a Webhook

```bash
# Get webhook ID first
WEBHOOK_ID=$(curl -s "${PORTAINER_URL}/api/webhooks" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq -r '.[] | select(.Token == "abc123-token-here") | .Id')

# Delete the webhook
curl -X DELETE "${PORTAINER_URL}/api/webhooks/${WEBHOOK_ID}" \
  -H "Authorization: Bearer ${API_TOKEN}"
```

## Security Best Practices for Webhooks

- Treat webhook URLs like passwords - they provide deployment access.
- Use HTTPS to prevent token interception.
- Rotate webhooks periodically by deleting and recreating them.
- Restrict webhook trigger IPs at the firewall or reverse proxy level.

## Conclusion

Portainer webhooks offer a lightweight, token-based trigger mechanism for automated deployments. They're simpler to use than full API credentials in CI/CD pipelines and can be shared with external systems without granting full Portainer API access.
