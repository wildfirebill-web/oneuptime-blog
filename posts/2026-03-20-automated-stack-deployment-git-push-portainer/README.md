# How to Set Up Automated Stack Deployment on Git Push with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, GitOps, Git, CI/CD, Docker, Automation, Webhook

Description: Learn how to configure Portainer to automatically redeploy stacks when you push changes to a Git repository using webhooks and GitOps workflows.

---

Portainer's GitOps integration lets you link a stack to a Git repository and automatically redeploy whenever you push changes. This means your deployed stack always reflects the latest version in your repo - no manual redeployments needed. This guide covers both Portainer's built-in Git integration and webhook-based CI/CD triggers.

---

## Approach 1: Portainer Native Git Integration

Portainer Business Edition supports native Git repository integration for stacks.

### Link a Stack to a Git Repository

1. In Portainer, go to **Stacks > Add Stack**
2. Select **Repository** as the build method
3. Fill in:
   - **Repository URL**: `https://github.com/yourorg/your-infrastructure.git`
   - **Repository reference**: `refs/heads/main`
   - **Compose file path**: `stacks/myapp/docker-compose.yml`
4. Optionally add a `.env` file for environment variables
5. Enable **Auto update** and set the polling interval (or use webhooks)
6. Click **Deploy the stack**

### Enable Git Webhook Trigger

In the stack settings, enable the webhook URL:
1. Go to the stack → **Settings**
2. Enable **GitOps update**
3. Copy the generated webhook URL
4. Add it to your GitHub repo: **Settings > Webhooks > Add webhook**

---

## Approach 2: GitHub Actions Webhook Trigger

Trigger Portainer stack redeployment from a GitHub Actions workflow.

```yaml
# .github/workflows/deploy.yml - deploy to Portainer on push to main

name: Deploy to Portainer

on:
  push:
    branches:
      - main
    paths:
      - "stacks/myapp/**"   # only trigger when stack files change

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Trigger Portainer redeploy via webhook
        run: |
          # Portainer webhook - triggers stack update with latest Git content
          curl -X POST \
            "${{ secrets.PORTAINER_WEBHOOK_URL }}" \
            --fail \
            --silent \
            --show-error
```

---

## Approach 3: Portainer API Stack Update from CI

For more control, use the Portainer API to update a stack from CI with a specific image tag.

```bash
#!/bin/bash
# deploy-to-portainer.sh - update a stack via Portainer API

PORTAINER_URL="https://portainer.example.com"
API_KEY="${PORTAINER_API_KEY}"
STACK_ID="${PORTAINER_STACK_ID}"
IMAGE_TAG="${GITHUB_SHA:-latest}"

echo "Deploying stack $STACK_ID with image tag: $IMAGE_TAG"

# Get current stack file content
STACK_FILE=$(curl -s -H "X-API-Key: $API_KEY" \
  "$PORTAINER_URL/api/stacks/$STACK_ID/file" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['StackFileContent'])")

# Update the stack with new env var (passes image tag to Compose)
curl -s -X PUT \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"StackFileContent\": $(echo "$STACK_FILE" | python3 -c "import sys,json; print(json.dumps(sys.stdin.read()))"),
    \"Env\": [
      {\"name\": \"IMAGE_TAG\", \"value\": \"$IMAGE_TAG\"}
    ],
    \"Prune\": true,
    \"PullImage\": true
  }" \
  "$PORTAINER_URL/api/stacks/$STACK_ID?endpointId=1"

echo "Stack deployment triggered."
```

---

## Docker Compose for GitOps Deployment

```yaml
# docker-compose.yml - uses IMAGE_TAG env var for automated deployments
version: "3.8"

services:
  webapp:
    image: myrepo/myapp:${IMAGE_TAG:-latest}
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      APP_ENV: production
      DEPLOYED_AT: ${DEPLOY_TIMESTAMP:-unknown}
```

---

## Summary

Automated stack deployment on Git push requires a Portainer stack linked to a Git repository and a mechanism to trigger updates on push - either Portainer's built-in webhook support (BE), a GitHub Actions workflow, or a direct API call. The cleanest approach for teams is Portainer's native GitOps integration: merge to main, and Portainer automatically pulls and redeploys.
