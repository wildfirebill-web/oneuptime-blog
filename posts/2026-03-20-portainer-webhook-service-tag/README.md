# How to Use Webhook Environment Variables (SERVICE_TAG) in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Webhooks, CI/CD, Automation

Description: Learn how to use the SERVICE_TAG environment variable with Portainer webhooks to deploy specific image versions through the same webhook URL.

## Introduction

Portainer's container webhooks are powerful because you can pass a `SERVICE_TAG` environment variable to override the image tag at deployment time. This means a single webhook URL can deploy any version of your application - just change the tag you pass. This is ideal for CI/CD pipelines where each build produces a new tagged image.

## Prerequisites

- Portainer Business Edition with container webhooks enabled
- A container with a webhook configured
- Understanding of Docker image tagging

## Understanding SERVICE_TAG

When Portainer receives a webhook trigger, it checks for a `SERVICE_TAG` environment variable in the request body. If present, it overrides the image tag for this deployment only.

```text
Normal webhook (uses container's current image tag):
POST /api/webhooks/YOUR-TOKEN
→ Pulls: myorg/myapp:latest (or whatever tag is configured)

With SERVICE_TAG:
POST /api/webhooks/YOUR-TOKEN
Body: {"env": [{"name": "SERVICE_TAG", "value": "v2.1.0"}]}
→ Pulls: myorg/myapp:v2.1.0
```

Note: This feature behavior may vary by Portainer version. Check your Portainer docs for the exact implementation.

## Method 1: Stack Webhook with SERVICE_TAG

For Portainer Stacks (more reliable), use the stack webhook with environment variables:

```bash
# Trigger a stack webhook with a specific version:

curl -X POST \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/webhooks/STACK-WEBHOOK-TOKEN" \
  -d '{"ServiceTag": "v2.1.0"}'
```

In the stack's docker-compose.yml, reference the environment variable:

```yaml
# docker-compose.yml for stack with tag support
version: "3.8"

services:
  app:
    # Reference SERVICE_TAG via env var substitution
    image: myorg/myapp:${SERVICE_TAG:-latest}
    restart: unless-stopped
    environment:
      - APP_VERSION=${SERVICE_TAG:-latest}
```

## Method 2: Using SERVICE_TAG in CI/CD

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Get version tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

      - name: Build and push image
        run: |
          docker build -t myorg/myapp:${{ steps.version.outputs.VERSION }} .
          docker push myorg/myapp:${{ steps.version.outputs.VERSION }}

      - name: Deploy via Portainer webhook
        run: |
          # For stack webhooks, SERVICE_TAG overrides the image tag
          curl -X POST \
            -H "Content-Type: application/json" \
            "${{ secrets.PORTAINER_WEBHOOK_URL }}" \
            -d "{\"ServiceTag\": \"${{ steps.version.outputs.VERSION }}\"}"
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - deploy

build:
  stage: build
  script:
    - docker build -t registry.gitlab.com/$CI_PROJECT_PATH:$CI_COMMIT_TAG .
    - docker push registry.gitlab.com/$CI_PROJECT_PATH:$CI_COMMIT_TAG
  only:
    - tags

deploy:
  stage: deploy
  script:
    - |
      curl -X POST \
        -H "Content-Type: application/json" \
        "${PORTAINER_WEBHOOK_URL}" \
        -d "{\"ServiceTag\": \"${CI_COMMIT_TAG}\"}"
  only:
    - tags
  environment:
    name: production
```

## Method 3: Version-Specific Webhooks

An alternative pattern: create a separate container (or stack env variable) per environment and use the webhook to deploy by tag:

```bash
#!/bin/bash
# deploy-version.sh
# Deploys a specific version using Portainer stack webhook

VERSION="${1:?Version required}"
WEBHOOK_URL="${PORTAINER_STACK_WEBHOOK_URL:?Webhook URL required}"

echo "Deploying version: ${VERSION}"

# Update the stack with the new version
RESPONSE=$(curl -s -w "\n%{http_code}" \
  -X POST \
  -H "Content-Type: application/json" \
  "${WEBHOOK_URL}" \
  -d "{\"ServiceTag\": \"${VERSION}\"}")

HTTP_CODE=$(echo "${RESPONSE}" | tail -1)
BODY=$(echo "${RESPONSE}" | head -1)

if [ "${HTTP_CODE}" == "200" ] || [ "${HTTP_CODE}" == "204" ]; then
    echo "✓ Deployment of version ${VERSION} triggered"
else
    echo "✗ Deployment failed: HTTP ${HTTP_CODE}"
    echo "Response: ${BODY}"
    exit 1
fi
```

## Method 4: Multiple Tags from One Pipeline

Deploy to staging with the commit SHA, and to production with the release tag:

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: myorg/myapp
          tags: |
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Deploy to staging
        run: |
          # Deploy SHA-tagged image to staging
          curl -X POST \
            -H "Content-Type: application/json" \
            "${{ secrets.PORTAINER_STAGING_WEBHOOK }}" \
            -d "{\"ServiceTag\": \"sha-${{ github.sha }}\"}"

  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/')

    steps:
      - name: Deploy to production
        run: |
          # Deploy version-tagged image to production
          TAG="${{ github.ref_name }}"
          curl -X POST \
            -H "Content-Type: application/json" \
            "${{ secrets.PORTAINER_PROD_WEBHOOK }}" \
            -d "{\"ServiceTag\": \"${TAG}\"}"
```

## Portainer Stack Environment Variable Approach

The most reliable pattern is to use stack environment variables and update them via the API:

```yaml
# docker-compose.yml
services:
  app:
    image: myorg/myapp:${APP_TAG:-latest}
    restart: unless-stopped
```

Then update the stack environment variable via Portainer API:

```bash
#!/bin/bash
# update-stack-tag.sh

PORTAINER_URL="${PORTAINER_URL}"
API_KEY="${PORTAINER_API_KEY}"
STACK_ID="${PORTAINER_STACK_ID}"
NEW_TAG="${1:?Tag required}"

# Get current stack config
STACK_INFO=$(curl -s \
  -H "X-API-Key: ${API_KEY}" \
  "${PORTAINER_URL}/api/stacks/${STACK_ID}")

# Update APP_TAG env var and redeploy
curl -s -X PUT \
  -H "X-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/stacks/${STACK_ID}" \
  -d "{
    \"env\": [{\"name\": \"APP_TAG\", \"value\": \"${NEW_TAG}\"}],
    \"prune\": false,
    \"pullImage\": true
  }"

echo "Stack updated to tag: ${NEW_TAG}"
```

## Conclusion

SERVICE_TAG with Portainer webhooks enables flexible, version-specific deployments through a single webhook URL. The stack-based approach is most powerful: use `${SERVICE_TAG:-latest}` in your compose file, pass the tag via webhook body or Portainer API, and achieve fully automated, version-controlled deployments in your CI/CD pipeline.
