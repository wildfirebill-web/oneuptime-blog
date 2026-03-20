# How to Trigger Webhooks via the Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Webhooks, CI/CD, Automation

Description: Learn how to create and trigger Portainer webhooks to automate container and service redeployments when new container images are pushed to a registry.

## Introduction

Portainer webhooks provide a simple HTTP endpoint that, when called, triggers a service or stack update - typically pulling the latest image and restarting the container. This is the simplest way to integrate Portainer with CI/CD pipelines, registry push events, and other automation triggers.

## Prerequisites

- Portainer CE or BE
- A running container or stack deployed in Portainer
- Network access from your CI/CD system to Portainer

## How Portainer Webhooks Work

1. You create a webhook in Portainer for a container or service
2. Portainer generates a unique URL
3. When that URL receives a POST request, Portainer:
   - Pulls the latest image for the container/service
   - Restarts the container/service with the new image
4. No authentication is required to trigger the webhook (the URL itself is the secret)

## Step 1: Create a Webhook for a Container

### Via Portainer UI

1. Go to **Containers**.
2. Click on the container name.
3. Scroll to the **Container webhook** section.
4. Click **Copy link** - this is your webhook URL.

The webhook URL format:
```text
https://portainer.example.com/api/webhooks/{token}
```

### Via Portainer API

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-auth-token"
ENDPOINT_ID=1
CONTAINER_ID="abc123def456"

# Create a webhook for a container

WEBHOOK=$(curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/webhooks" \
  -d "{
    \"resourceId\": \"$CONTAINER_ID\",
    \"endpointId\": $ENDPOINT_ID,
    \"webhookType\": 1
  }")

WEBHOOK_TOKEN=$(echo $WEBHOOK | jq -r '.token')
echo "Webhook created!"
echo "Webhook URL: ${PORTAINER_URL}/api/webhooks/${WEBHOOK_TOKEN}"
```

## Step 2: Create a Webhook for a Service (Swarm)

```bash
SERVICE_ID="your-swarm-service-id"

# Create a webhook for a Docker Swarm service
WEBHOOK=$(curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/webhooks" \
  -d "{
    \"resourceId\": \"$SERVICE_ID\",
    \"endpointId\": $ENDPOINT_ID,
    \"webhookType\": 2
  }")

WEBHOOK_TOKEN=$(echo $WEBHOOK | jq -r '.token')
echo "Service webhook URL: ${PORTAINER_URL}/api/webhooks/${WEBHOOK_TOKEN}"
```

## Step 3: Trigger a Webhook

```bash
WEBHOOK_TOKEN="your-webhook-token"

# Trigger the webhook (POST to the webhook URL)
curl -s -X POST \
  "${PORTAINER_URL}/api/webhooks/${WEBHOOK_TOKEN}"

# Expected response: 204 No Content on success
echo "Webhook triggered. Container redeployment initiated."
```

## Step 4: List All Webhooks

```bash
# List all webhooks
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/webhooks" | jq '.[] | {
    id: .Id,
    token: .Token,
    resourceId: .ResourceID,
    type: .WebhookType
  }'
```

## Step 5: Delete a Webhook

```bash
WEBHOOK_ID=3

# Delete a webhook
curl -s -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/webhooks/${WEBHOOK_ID}"

echo "Webhook deleted."
```

## Step 6: Integrate Webhook with GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        run: |
          docker build -t myregistry.io/myapp:latest .
          docker push myregistry.io/myapp:latest

      - name: Trigger Portainer redeployment
        run: |
          # Trigger the Portainer webhook to pull new image and restart
          HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
            "${{ secrets.PORTAINER_WEBHOOK_URL }}")

          if [ "$HTTP_STATUS" = "204" ]; then
            echo "Deployment triggered successfully!"
          else
            echo "Webhook trigger failed with status: $HTTP_STATUS"
            exit 1
          fi
```

## Step 7: Integrate with Docker Hub Webhooks

Configure Docker Hub to automatically trigger Portainer redeployment after a push:

1. Log into Docker Hub.
2. Navigate to your repository → **Webhooks**.
3. Click **Create a Webhook**.
4. Enter your Portainer webhook URL:
   ```text
   https://portainer.example.com/api/webhooks/{your-token}
   ```
5. Click **Create Webhook**.

Now every `docker push` to your Docker Hub repository will automatically trigger Portainer to pull the new image and restart the container.

## Step 8: Trigger Webhook with Stack Update

For stacks, use the stack's webhook or trigger a stack update programmatically:

```bash
STACK_ID=3

# Get the stack webhook URL
STACK=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/stacks/${STACK_ID}")

WEBHOOK_ID=$(echo $STACK | jq -r '.AutoUpdate.Webhook // empty')

if [ -n "$WEBHOOK_ID" ]; then
  echo "Stack webhook: ${PORTAINER_URL}/api/webhooks/${WEBHOOK_ID}"
else
  # Create a webhook for the stack via git-connected stacks
  echo "This stack doesn't have an auto-update webhook configured."
  echo "Configure Git-based stack with webhook option in Portainer."
fi
```

## Security Considerations

Webhook URLs are essentially bearer tokens. Protect them by:

- **Storing in secrets managers** (GitHub Secrets, Vault, etc.)
- **Using HTTPS** for all webhook calls
- **Rotating periodically**: Delete and recreate webhooks when team members leave
- **Limiting exposure**: Don't log webhook URLs or expose them in build artifacts

## Conclusion

Portainer webhooks provide a dead-simple integration point for automated redeployments. Create a webhook for your container or service, store the URL securely in your CI/CD system, and call it after every successful image push. This pattern enables continuous deployment without complex pipeline integrations - a simple HTTP POST is all it takes to trigger a rolling update.
