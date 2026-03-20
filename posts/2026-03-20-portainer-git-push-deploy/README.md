# How to Set Up Automated Stack Deployment on Git Push with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, GitOps, Automation, CI/CD, Docker

Description: Configure automatic stack deployment in Portainer triggered by Git push events, enabling GitOps workflows for container deployments.

## Introduction

GitOps makes your Git repository the source of truth for infrastructure state. With Portainer's Git integration and webhook support, pushes to specific branches automatically trigger stack deployments—no manual Portainer UI interaction required after the initial setup.

## Method 1: Portainer Git Integration with Polling

Portainer's built-in Git polling checks for changes on a schedule:

```bash
# In Portainer: Stacks > Add Stack > Repository
# Configure:
# - Repository URL: https://github.com/yourorg/your-repo
# - Repository reference: refs/heads/main
# - Compose file path: docker/docker-compose.yml
# - Enable "Auto update" with interval: 5 minutes

# For private repos, add credentials:
# Username: your-github-username
# Password/Token: your-personal-access-token
```

## Method 2: Portainer Webhooks (Instant Deployment)

Portainer provides a webhook URL that triggers immediate re-deployment:

```bash
# In Portainer: Stacks > Your Stack > Webhooks > Enable
# Copy the webhook URL:
# https://portainer.example.com/api/webhooks/WEBHOOK_TOKEN

# Test it manually
curl -X POST "https://portainer.example.com/api/webhooks/YOUR_WEBHOOK_TOKEN"
```

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
    paths:
      - 'app/**'
      - 'docker/**'
      - 'docker-compose.yml'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build and push Docker image
        run: |
          docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_TOKEN }}
          docker build -t myapp:${{ github.sha }} .
          docker tag myapp:${{ github.sha }} myapp:latest
          docker push myapp:${{ github.sha }}
          docker push myapp:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Portainer deployment
        run: |
          # Update stack with new image tag
          curl -X POST \
            -H "Content-Type: application/json" \
            "https://portainer.example.com/api/webhooks/${{ secrets.PORTAINER_WEBHOOK_TOKEN }}"
          
          echo "Deployment triggered via Portainer webhook"
      
      - name: Wait and verify deployment
        run: |
          sleep 30  # Wait for deployment to complete
          
          # Check if stack is healthy
          STATUS=$(curl -s \
            -H "X-API-Key: ${{ secrets.PORTAINER_API_KEY }}" \
            "https://portainer.example.com/api/stacks" \
            | python3 -c "
import sys, json
stacks = json.load(sys.stdin)
for s in stacks:
    if s['Name'] == 'production-app':
        print('running' if s['Status'] == 1 else 'stopped')
        break
")
          
          if [ "$STATUS" != "running" ]; then
            echo "Deployment failed! Stack is not running."
            exit 1
          fi
          echo "Deployment successful!"
```

### GitLab CI Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

build:
  stage: build
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE

deploy_production:
  stage: deploy
  environment: production
  only:
    - main
  script:
    # Update the image tag in the compose file
    - |
      cat docker-compose.yml | \
        sed "s|image: myapp:.*|image: $DOCKER_IMAGE|" > /tmp/updated-compose.yml
    
    # Deploy via Portainer API
    - |
      COMPOSE_CONTENT=$(cat /tmp/updated-compose.yml | python3 -c "import sys,json; print(json.dumps(sys.stdin.read()))")
      curl -X PUT \
        -H "X-API-Key: $PORTAINER_API_KEY" \
        -H "Content-Type: application/json" \
        -d "{
          \"StackFileContent\": $COMPOSE_CONTENT,
          \"Env\": [{\"name\": \"IMAGE_TAG\", \"value\": \"$CI_COMMIT_SHA\"}],
          \"Prune\": false
        }" \
        "$PORTAINER_URL/api/stacks/$STACK_ID?endpointId=$ENDPOINT_ID"
```

## Method 3: Portainer Git Stack with Auto-Update

```bash
# Deploy a stack linked to a Git repository
curl -X POST \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "production-app",
    "RepositoryURL": "https://github.com/yourorg/your-app",
    "RepositoryReferenceName": "refs/heads/main",
    "ComposeFilePathInRepository": "docker/docker-compose.yml",
    "RepositoryAuthentication": true,
    "RepositoryUsername": "your-username",
    "RepositoryPassword": "your-token",
    "AutoUpdate": {
      "Interval": "5m",
      "Webhook": "true"
    }
  }' \
  "https://portainer.example.com/api/stacks/create/standalone/repository?endpointId=1"
```

## Rollback on Failure

```bash
# GitHub Actions with automatic rollback
- name: Verify and rollback if needed
  run: |
    # Wait for health check
    for i in {1..10}; do
      STATUS=$(curl -sf https://myapp.example.com/health | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])" 2>/dev/null)
      if [ "$STATUS" = "ok" ]; then
        echo "Deployment healthy!"
        exit 0
      fi
      echo "Waiting for health check... attempt $i"
      sleep 10
    done
    
    echo "Health check failed! Rolling back..."
    # Trigger previous version deployment via Portainer
    curl -X POST \
      -H "X-API-Key: ${{ secrets.PORTAINER_API_KEY }}" \
      -H "Content-Type: application/json" \
      -d "{\"StackFileContent\": \"$(cat docker-compose.prev.yml | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read()))')\"}" \
      "${{ secrets.PORTAINER_URL }}/api/stacks/${{ secrets.STACK_ID }}?endpointId=1"
    exit 1
```

## Conclusion

Automated Git-push deployment with Portainer creates a seamless GitOps workflow where code changes automatically flow to running containers. Portainer's webhook and Git polling mechanisms offer flexibility to choose between instant or scheduled deployments. Combined with CI/CD pipelines for testing and image building, this creates a complete automated delivery pipeline.
