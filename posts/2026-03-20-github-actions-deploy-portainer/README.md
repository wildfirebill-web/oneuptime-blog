# How to Use GitHub Actions to Deploy to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, GitHub Actions, CI/CD, Docker, Deployment, Automation

Description: Learn how to create GitHub Actions workflows that build Docker images and deploy them to Portainer stacks automatically on push.

---

GitHub Actions can trigger Portainer deployments via the Portainer API or stack webhooks. This guide covers a complete workflow that builds, tests, and deploys to Portainer.

## Basic Webhook Deployment Workflow

The simplest approach: trigger a Portainer stack webhook after pushing a new image.

Create `.github/workflows/deploy.yml`:

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to Portainer
        run: |
          curl -fsS -X POST "${{ secrets.PORTAINER_WEBHOOK_URL }}"
```

## Full Pipeline with Staging and Production

A more complete workflow with environment promotion:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha
            type=ref,event=branch
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    if: github.ref == 'refs/heads/develop' || github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to staging via Portainer API
        run: |
          TOKEN=$(curl -s -X POST "${{ secrets.PORTAINER_URL }}/api/auth" \
            -H "Content-Type: application/json" \
            -d '{"Username":"${{ secrets.PORTAINER_USER }}","Password":"${{ secrets.PORTAINER_PASSWORD }}"}' \
            | jq -r .jwt)

          STACK_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
            "${{ secrets.PORTAINER_URL }}/api/stacks" | \
            jq -r '.[] | select(.Name=="my-app-staging") | .Id')

          curl -fsS -X POST \
            -H "Authorization: Bearer $TOKEN" \
            "${{ secrets.PORTAINER_URL }}/api/stacks/${STACK_ID}/images/update?pullImage=true"

  smoke-test:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run smoke tests
        run: |
          sleep 30  # Wait for containers to start
          curl -fsS https://staging.example.com/health

  deploy-production:
    needs: smoke-test
    runs-on: ubuntu-latest
    environment: production   # Requires approval in GitHub environment settings
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to production
        run: |
          curl -fsS -X POST "${{ secrets.PORTAINER_PROD_WEBHOOK_URL }}"
```

## Storing Secrets in GitHub

Add secrets to your repository under **Settings > Secrets and variables > Actions**:

| Secret Name | Description |
|-------------|-------------|
| `PORTAINER_URL` | Portainer instance URL |
| `PORTAINER_USER` | Portainer admin username |
| `PORTAINER_PASSWORD` | Portainer admin password |
| `PORTAINER_PROD_WEBHOOK_URL` | Stack webhook for production |

## Using GitHub Environments for Approvals

Configure required reviewers for the `production` environment:

1. Go to **Settings > Environments > production**.
2. Enable **Required reviewers** and add team members.
3. The `deploy-production` job will pause and request approval before running.

## Rollback Workflow

Add a manual rollback workflow triggered from the GitHub Actions UI:

```yaml
name: Rollback

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to roll back to'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - name: Update stack and redeploy
        run: |
          # Update the stack compose file with the rollback tag
          # then call the Portainer API to redeploy
          echo "Rolling back to ${{ github.event.inputs.image_tag }}"
```
