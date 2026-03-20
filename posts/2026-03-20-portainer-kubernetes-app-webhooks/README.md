# How to Set Up Application Webhooks in Portainer for Kubernetes - App

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Webhooks, CI/CD, DevOps

Description: Learn how to configure Portainer application webhooks for Kubernetes to enable automated deployments triggered by CI/CD pipelines.

## Introduction

Portainer application webhooks for Kubernetes work the same way as for Docker - an HTTP POST to the webhook URL triggers Portainer to update the application. This enables push-based CI/CD where your build pipeline triggers Portainer to redeploy after pushing a new image. This guide covers setting up and using Kubernetes application webhooks in Portainer.

## Prerequisites

- Portainer BE with Kubernetes environment (webhooks are a BE feature for Kubernetes)
- A running Kubernetes application in Portainer
- A CI/CD system (GitHub Actions, GitLab CI, Jenkins, etc.)

## Step 1: Enable Webhook for a Kubernetes Application

1. Navigate to your Kubernetes environment in Portainer
2. Click **Applications**
3. Click on the application you want to automate
4. Find the **Webhooks** section
5. Toggle **Enable webhook** to ON
6. Copy the generated webhook URL:

```text
https://portainer.example.com:9443/api/webhooks/abc123def456...
```

## Step 2: Test the Webhook

```bash
# Test webhook call

curl -X POST \
  "https://portainer.example.com:9443/api/webhooks/abc123def456" \
  --insecure \
  --max-time 30

# Expected response: HTTP 200
# Portainer will pull the latest image version and update the deployment
```

## Step 3: Understand Webhook Behavior

When a webhook is triggered:

1. Portainer identifies the Kubernetes deployment
2. Pulls the latest version of the configured image tag from the registry
3. Updates the deployment's container image
4. Kubernetes performs a rolling update

**Important:** The webhook pulls the same image tag that's configured in the deployment. If using `latest`, it pulls the latest image at `latest`. For version-pinned deployments, pushing a new image with the same tag triggers a redeploy.

## Step 4: Integrate with GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Build, Push, and Deploy

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
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
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Trigger Portainer deployment
        run: |
          curl -X POST \
            "${{ secrets.PORTAINER_WEBHOOK_URL }}" \
            --fail \
            --max-time 60 \
            --retry 3

      - name: Verify deployment
        run: |
          # Optional: verify the deployment rolled out successfully
          sleep 30
          curl -sk \
            "${{ secrets.PORTAINER_API_URL }}/api/endpoints/1/kubernetes/apis/apps/v1/namespaces/production/deployments/my-app" \
            -H "X-API-Key: ${{ secrets.PORTAINER_API_KEY }}" | \
            jq '.status.readyReplicas'
```

## Step 5: Integrate with GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - deploy

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main

deploy-production:
  stage: deploy
  image: curlimages/curl:latest
  script:
    - |
      curl -X POST "$PORTAINER_WEBHOOK_URL" \
        --fail \
        --max-time 60 \
        --retry 3 \
        --retry-delay 10
  environment:
    name: production
    url: https://myapp.example.com
  only:
    - main
```

## Step 6: Use Portainer API for More Control

For more complex deployment automation, use the Portainer API directly:

```bash
# Get API key from Portainer: Settings → Account → Access tokens
PORTAINER_API_KEY="ptr_xxxxxxxxxxxxx"
PORTAINER_URL="https://portainer.example.com:9443"
ENVIRONMENT_ID="1"    # Your Kubernetes environment ID
NAMESPACE="production"
DEPLOYMENT="my-app"
NEW_IMAGE="registry.company.com/my-app:v2.1.0"

# Update deployment image via Portainer API
curl -X PUT \
  "$PORTAINER_URL/api/endpoints/$ENVIRONMENT_ID/kubernetes/apis/apps/v1/namespaces/$NAMESPACE/deployments/$DEPLOYMENT" \
  -H "X-API-Key: $PORTAINER_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE -o json | \
    jq --arg img "$NEW_IMAGE" '.spec.template.spec.containers[0].image = $img')"
```

## Step 7: Secure Webhook URLs

Protect webhook URLs like secrets:

```bash
# Store in CI/CD secrets (never in code)
# GitHub: Settings → Secrets and variables → Actions → New repository secret
# Name: PORTAINER_WEBHOOK_URL
# Value: https://portainer.example.com:9443/api/webhooks/abc123...
```

Best practices:
- Never commit webhook URLs to source code
- Rotate webhook URLs periodically
- Restrict Portainer API access by IP at the firewall level

## Step 8: Handle Webhook Failures

Add error handling in your CI/CD pipeline:

```bash
# GitHub Actions step with proper error handling
- name: Deploy to Kubernetes via Portainer
  run: |
    RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" \
      -X POST "${{ secrets.PORTAINER_WEBHOOK_URL }}" \
      --max-time 60)

    if [ "$RESPONSE" -ne 200 ]; then
      echo "ERROR: Portainer webhook returned HTTP $RESPONSE"
      exit 1
    fi
    echo "Deployment triggered successfully (HTTP $RESPONSE)"
```

## Conclusion

Portainer webhooks for Kubernetes applications bridge your CI/CD pipeline and your Kubernetes deployments. By storing the webhook URL as a CI/CD secret and calling it after pushing a new image, you achieve automated, event-driven deployments without exposing Portainer credentials to your build systems. This pattern scales well across multiple applications and environments.
