# How to Set Up CI/CD with Portainer and GitLab CI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, CI/CD, GitLab, Docker, Automation

Description: Learn how to configure GitLab CI/CD pipelines to build Docker images and automatically deploy them to Portainer-managed environments using webhooks and the Portainer API.

## Introduction

GitLab CI/CD combined with Portainer creates a fully self-hosted CI/CD pipeline. GitLab builds and pushes images to GitLab Container Registry or your private registry, then triggers Portainer to redeploy. This guide covers setting up this pipeline with practical `.gitlab-ci.yml` examples.

## Prerequisites

- Portainer CE or BE with a Docker environment
- GitLab instance (cloud or self-hosted) with CI/CD enabled
- Portainer webhook URL or API key
- Docker runner configured in GitLab

## Architecture

```
Git push → GitLab CI → Build & Test → Push to GitLab Registry → Deploy to Portainer
```

## Step 1: Configure GitLab CI/CD Variables

In your GitLab project → **Settings** → **CI/CD** → **Variables**, add:

| Variable | Value | Protected | Masked |
|----------|-------|-----------|--------|
| `PORTAINER_URL` | https://portainer.example.com | Yes | No |
| `PORTAINER_WEBHOOK_URL` | Webhook URL | Yes | Yes |
| `PORTAINER_API_KEY` | API access token | Yes | Yes |
| `PORTAINER_ENDPOINT_ID` | 1 | No | No |

## Step 2: Create the GitLab CI Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_NAME: $CI_REGISTRY_IMAGE
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

# ===== Test Stage =====
test:
  stage: test
  image: node:20-alpine
  script:
    - npm ci
    - npm test
  only:
    - merge_requests
    - main

# ===== Build Stage =====
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_TAG -t $IMAGE_NAME:latest .
    - docker push $IMAGE_NAME:$IMAGE_TAG
    - docker push $IMAGE_NAME:latest
    - echo "IMAGE_TAG=$IMAGE_TAG" >> build.env
  artifacts:
    reports:
      dotenv: build.env
  only:
    - main

# ===== Deploy Stage =====
deploy:
  stage: deploy
  image: curlimages/curl:latest
  needs:
    - job: build
      artifacts: true
  script:
    - |
      echo "Deploying image $IMAGE_NAME:$IMAGE_TAG to Portainer..."

      HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
        -X POST "$PORTAINER_WEBHOOK_URL")

      echo "Webhook response status: $HTTP_STATUS"

      if [ "$HTTP_STATUS" = "200" ] || [ "$HTTP_STATUS" = "204" ]; then
        echo "Deployment triggered successfully!"
      else
        echo "ERROR: Webhook failed with status $HTTP_STATUS"
        exit 1
      fi
  only:
    - main
  environment:
    name: production
    url: https://myapp.example.com
```

## Step 3: Advanced Pipeline with API-Based Deployment

For more control, update the stack with the specific image tag:

```yaml
# .gitlab-ci.yml — Advanced version
deploy-advanced:
  stage: deploy
  image: alpine:3.19
  before_script:
    - apk add --no-cache curl jq
  script:
    - |
      STACK_NAME="my-app"

      echo "Looking up stack '$STACK_NAME'..."
      STACK_ID=$(curl -sf -H "X-API-Key: $PORTAINER_API_KEY" \
        "$PORTAINER_URL/api/stacks" | \
        jq -r --arg n "$STACK_NAME" '.[] | select(.Name == $n) | .Id')

      if [ -z "$STACK_ID" ]; then
        echo "ERROR: Stack '$STACK_NAME' not found"
        exit 1
      fi

      echo "Updating stack $STACK_ID with image tag $IMAGE_TAG..."

      # Get current compose content
      COMPOSE=$(curl -sf -H "X-API-Key: $PORTAINER_API_KEY" \
        "$PORTAINER_URL/api/stacks/$STACK_ID/file" | jq -r '.StackFileContent')

      # Update the stack
      RESPONSE=$(curl -s -X PUT \
        -H "X-API-Key: $PORTAINER_API_KEY" \
        -H "Content-Type: application/json" \
        "$PORTAINER_URL/api/stacks/$STACK_ID?endpointId=$PORTAINER_ENDPOINT_ID" \
        -d "{
          \"stackFileContent\": $(echo "$COMPOSE" | jq -Rs .),
          \"env\": [{\"name\": \"IMAGE_TAG\", \"value\": \"$IMAGE_TAG\"}],
          \"pullImage\": true,
          \"prune\": false
        }")

      echo "Update response: $RESPONSE"
      echo "Deployment complete for version $IMAGE_TAG"
  only:
    - main
  needs:
    - job: build
      artifacts: true
```

## Step 4: Multi-Environment Pipeline

Deploy to staging on feature branches, production on main:

```yaml
# Multi-environment deployment
.deploy_template: &deploy_template
  stage: deploy
  image: alpine:3.19
  before_script:
    - apk add --no-cache curl jq
  script:
    - |
      echo "Deploying to $CI_ENVIRONMENT_NAME..."
      curl -s -X POST \
        -H "X-API-Key: $PORTAINER_API_KEY" \
        -H "Content-Type: application/json" \
        "$PORTAINER_URL/api/stacks/$STACK_ID?endpointId=$ENDPOINT_ID" \
        -d "{
          \"stackFileContent\": $(cat docker-compose.yml | jq -Rs .),
          \"env\": [{\"name\": \"IMAGE_TAG\", \"value\": \"$IMAGE_TAG\"}],
          \"pullImage\": true
        }"

deploy-staging:
  <<: *deploy_template
  variables:
    STACK_ID: "2"
    ENDPOINT_ID: "2"
  only:
    - develop
  environment:
    name: staging
    url: https://staging.example.com

deploy-production:
  <<: *deploy_template
  variables:
    STACK_ID: "1"
    ENDPOINT_ID: "1"
  only:
    - main
  when: manual  # Require manual approval for production
  environment:
    name: production
    url: https://myapp.example.com
```

## Step 5: Health Check in Pipeline

```yaml
verify-deployment:
  stage: deploy
  image: alpine:3.19
  needs:
    - deploy-production
  script:
    - apk add --no-cache curl
    - |
      echo "Verifying deployment health..."
      MAX_RETRIES=12
      for i in $(seq 1 $MAX_RETRIES); do
        if curl -sf "https://myapp.example.com/health" > /dev/null 2>&1; then
          echo "Application is healthy!"
          exit 0
        fi
        echo "Attempt $i/$MAX_RETRIES — waiting 10s..."
        sleep 10
      done
      echo "ERROR: Health check failed after 2 minutes"
      exit 1
  only:
    - main
```

## Conclusion

GitLab CI/CD and Portainer together provide a powerful, self-hostable CI/CD stack. GitLab's built-in Container Registry eliminates the need for an external registry, while Portainer handles container orchestration. Use the webhook approach for simplicity and the API approach when you need to pass dynamic environment variables like image tags to your deployed stacks.
