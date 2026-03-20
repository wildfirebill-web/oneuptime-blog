# How to Set Up CI/CD with Portainer and GitHub Actions - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, CI/CD, GitHub Actions, Docker, Automation

Description: Learn how to build a complete CI/CD pipeline using GitHub Actions and Portainer to automatically build, push, and deploy Docker containers on every code push.

## Introduction

GitHub Actions combined with Portainer creates a powerful, free CI/CD pipeline: GitHub Actions builds and pushes your container image, then triggers Portainer to redeploy your stack with the new image. This guide walks through setting up this complete pipeline.

## Prerequisites

- Portainer CE or BE with a Docker environment
- A GitHub repository with your application code
- A container registry (Docker Hub, GHCR, or private registry)
- Portainer webhook URL for your stack (or API key for more control)

## Architecture

```text
Push to main → GitHub Actions → Build image → Push to registry → Trigger Portainer webhook → Container updated
```

## Step 1: Create Your Docker Compose Stack in Portainer

First, deploy a stack in Portainer that references your container image:

```yaml
# docker-compose.yml (in your GitHub repo)

version: "3.8"
services:
  app:
    image: ghcr.io/your-org/your-app:${IMAGE_TAG:-latest}
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
```

Deploy this via Portainer and note the stack name.

## Step 2: Set Up a Portainer Webhook

1. In Portainer, click on your stack.
2. Enable **Container webhook** (for individual containers) or deploy via Git repository with webhook.
3. Copy the webhook URL.

## Step 3: Configure GitHub Repository Secrets

In your GitHub repository → **Settings** → **Secrets and variables** → **Actions**, add:

| Secret | Value |
|--------|-------|
| `PORTAINER_WEBHOOK_URL` | Your Portainer webhook URL |
| `PORTAINER_URL` | https://portainer.example.com |
| `PORTAINER_API_KEY` | Your Portainer API access token |
| `REGISTRY_USERNAME` | Registry username |
| `REGISTRY_TOKEN` | Registry access token |

## Step 4: Create the GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.version }}

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log into container registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=sha-
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
      - name: Trigger Portainer redeployment
        run: |
          echo "Triggering Portainer deployment..."

          HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            -X POST "${{ secrets.PORTAINER_WEBHOOK_URL }}")

          echo "Webhook response: $HTTP_STATUS"

          if [ "$HTTP_STATUS" = "200" ] || [ "$HTTP_STATUS" = "204" ]; then
            echo "Deployment triggered successfully!"
          else
            echo "ERROR: Deployment trigger failed with status $HTTP_STATUS"
            exit 1
          fi

      - name: Wait for deployment to complete
        run: |
          echo "Waiting 30 seconds for deployment to stabilize..."
          sleep 30
          echo "Deployment complete."
```

## Step 5: Advanced Deployment with Stack Updates

For more control, use the Portainer API to update the stack's environment variables with the new image tag:

```yaml
# .github/workflows/deploy-advanced.yml
name: Advanced Build and Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Get short SHA
        id: sha
        run: echo "SHORT_SHA=${GITHUB_SHA::8}" >> $GITHUB_OUTPUT

      - name: Build and push
        run: |
          IMAGE_TAG="sha-${{ steps.sha.outputs.SHORT_SHA }}"
          docker build -t ghcr.io/${{ github.repository }}:${IMAGE_TAG} .
          docker push ghcr.io/${{ github.repository }}:${IMAGE_TAG}
          echo "IMAGE_TAG=${IMAGE_TAG}" >> $GITHUB_ENV

      - name: Update Portainer stack with new image tag
        env:
          PORTAINER_URL: ${{ secrets.PORTAINER_URL }}
          PORTAINER_API_KEY: ${{ secrets.PORTAINER_API_KEY }}
          STACK_NAME: my-app
          ENDPOINT_ID: 1
        run: |
          # Get stack ID by name
          STACK_ID=$(curl -s -H "X-API-Key: $PORTAINER_API_KEY" \
            "$PORTAINER_URL/api/stacks" | \
            jq -r --arg n "$STACK_NAME" '.[] | select(.Name == $n) | .Id')

          # Get current stack content
          STACK_CONTENT=$(curl -s -H "X-API-Key: $PORTAINER_API_KEY" \
            "$PORTAINER_URL/api/stacks/$STACK_ID/file" | jq -r '.StackFileContent')

          # Update stack with new IMAGE_TAG
          curl -s -X PUT -H "X-API-Key: $PORTAINER_API_KEY" \
            -H "Content-Type: application/json" \
            "$PORTAINER_URL/api/stacks/$STACK_ID?endpointId=$ENDPOINT_ID" \
            -d "{
              \"stackFileContent\": $(echo "$STACK_CONTENT" | jq -Rs .),
              \"env\": [{\"name\": \"IMAGE_TAG\", \"value\": \"$IMAGE_TAG\"}],
              \"pullImage\": true
            }"

          echo "Stack updated with image tag: $IMAGE_TAG"
```

## Step 6: Add Health Check Verification

```yaml
      - name: Verify deployment health
        env:
          APP_URL: https://myapp.example.com
        run: |
          echo "Waiting for application to be healthy..."
          MAX_RETRIES=12  # 12 * 10s = 2 minutes
          RETRY=0

          until curl -sf "$APP_URL/health" > /dev/null 2>&1; do
            RETRY=$((RETRY + 1))
            if [ $RETRY -ge $MAX_RETRIES ]; then
              echo "ERROR: Application health check failed after 2 minutes"
              exit 1
            fi
            echo "Attempt $RETRY/$MAX_RETRIES - waiting 10s..."
            sleep 10
          done

          echo "Application is healthy!"
```

## Conclusion

GitHub Actions and Portainer form a complete, cost-effective CI/CD pipeline. GitHub Actions handles building and pushing images, while Portainer manages container lifecycle. The webhook trigger is the simplest integration; the API-based approach gives you more control for complex deployments with dynamic image tags, environment variable injection, and multi-stack orchestration.
