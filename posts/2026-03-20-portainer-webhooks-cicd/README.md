# How to Use Portainer Webhooks in CI/CD Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Webhooks, CI/CD, Automation, GitOps

Description: Configure and use Portainer webhooks to trigger stack redeployments from CI/CD pipelines, GitHub, GitLab, and custom scripts.

## Introduction

Portainer webhooks provide a simple HTTP endpoint that triggers a stack or service update when called. This is the simplest way to integrate Portainer with CI/CD pipelines — your pipeline pushes a new image and calls the webhook, Portainer pulls the latest image and restarts the service. No Portainer API key required.

## Step 1: Configure Portainer Stack Webhooks

### For Docker Compose Stacks:
1. In Portainer, navigate to **Stacks**
2. Click your stack name
3. Scroll to **Stack Webhooks**
4. Toggle **Enable stack webhook** to ON
5. Copy the webhook URL

### For Docker Swarm Services:
1. Navigate to **Services**
2. Click your service
3. Find the **Service webhook** section
4. Copy the webhook URL

The URL format: `https://portainer.yourdomain.com/api/webhooks/{uuid}`

## Step 2: Test the Webhook

```bash
# Test webhook with curl (no authentication required)
curl -X POST "https://portainer.yourdomain.com/api/webhooks/your-webhook-uuid"

# Expected response: 200 OK (empty body)
# Portainer will pull the latest image and redeploy
```

## Step 3: Integrate with GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy on Push

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: registry.yourdomain.com/myapp:latest

      - name: Trigger Portainer deployment
        run: |
          HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            -X POST "${{ secrets.PORTAINER_WEBHOOK_URL }}")

          if [ "$HTTP_STATUS" = "200" ]; then
            echo "Deployment triggered successfully"
          else
            echo "Webhook failed with HTTP $HTTP_STATUS"
            exit 1
          fi
```

## Step 4: Integrate with GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - deploy

build:
  stage: build
  script:
    - docker build -t registry.yourdomain.com/myapp:latest .
    - docker push registry.yourdomain.com/myapp:latest

deploy:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache curl
  script:
    - |
      HTTP=$(curl -s -o /dev/null -w "%{http_code}" \
        -X POST "$PORTAINER_WEBHOOK_URL")

      [ "$HTTP" = "200" ] && echo "Deployed!" || exit 1
  environment:
    name: production
  only:
    - main
```

## Step 5: Integrate with Jenkins

```groovy
// Jenkinsfile
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'docker build -t registry.yourdomain.com/myapp:latest .'
                sh 'docker push registry.yourdomain.com/myapp:latest'
            }
        }

        stage('Deploy via Webhook') {
            steps {
                script {
                    def response = httpRequest(
                        url: "${PORTAINER_WEBHOOK_URL}",
                        httpMode: 'POST',
                        validResponseCodes: '200'
                    )
                    echo "Deployment triggered: ${response.status}"
                }
            }
        }
    }
}
```

## Step 6: Webhook with Image Update Verification

```bash
#!/bin/bash
# deploy-and-verify.sh - Deploy via webhook and verify

WEBHOOK_URL="${1:?Provide webhook URL}"
HEALTH_URL="${2:?Provide health check URL}"

echo "Triggering deployment..."
HTTP=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$WEBHOOK_URL")

if [ "$HTTP" != "200" ]; then
    echo "Webhook failed: HTTP $HTTP"
    exit 1
fi

echo "Webhook triggered. Waiting for deployment..."

# Wait for deployment to complete
for i in $(seq 1 30); do
    sleep 5
    HTTP=$(curl -s -o /dev/null -w "%{http_code}" "$HEALTH_URL")
    if [ "$HTTP" = "200" ]; then
        echo "Deployment verified! Service is healthy."
        exit 0
    fi
    echo "Attempt $i/30: HTTP $HTTP"
done

echo "TIMEOUT: Service not healthy after 150 seconds"
exit 1
```

## Step 7: Multiple Webhooks for Multi-Environment Deployment

```bash
#!/bin/bash
# multi-env-deploy.sh - Deploy to multiple environments

STAGING_WEBHOOK="${STAGING_WEBHOOK:?Required}"
PROD_WEBHOOK="${PROD_WEBHOOK:?Required}"

# Always deploy to staging
echo "Deploying to staging..."
curl -s -X POST "$STAGING_WEBHOOK"

# Verify staging
sleep 30
STAGING_HEALTH=$(curl -s -o /dev/null -w "%{http_code}" \
    "https://staging.yourdomain.com/health")

if [ "$STAGING_HEALTH" != "200" ]; then
    echo "Staging unhealthy. Blocking production deployment."
    exit 1
fi

echo "Staging healthy. Deploying to production..."
curl -s -X POST "$PROD_WEBHOOK"

echo "Production deployment triggered!"
```

## Step 8: Secure Webhooks with a Proxy

By default, webhooks require no authentication. For additional security, put Portainer behind a proxy that adds IP allowlisting:

```nginx
# nginx.conf - Restrict webhook access to CI/CD servers
location ~* ^/api/webhooks/ {
    # Only allow GitHub Actions, GitLab CI, and your CI server
    allow 140.82.112.0/20;  # GitHub Actions IPs
    allow 34.74.90.64/28;   # GitLab.com CI
    allow 10.0.0.100;        # Your internal CI server
    deny all;

    proxy_pass http://portainer:9000;
    proxy_set_header Host $host;
}
```

## Monitoring Webhook Deployments in Portainer

After a webhook triggers:
1. Go to **Stacks** or **Services** to see the update
2. Container list will show containers being restarted
3. Check **Logs** to verify the new version started correctly

```bash
# Verify new image is running after webhook
docker inspect my_container | jq '.[].Config.Image'

# Check when container last restarted
docker inspect my_container | jq '.[].State.StartedAt'
```

## Conclusion

Portainer webhooks provide the simplest possible CI/CD integration — a single HTTP POST triggers a full stack redeployment. The URL contains a secret UUID, making it reasonably secure without requiring API key management in your pipelines. Combine webhooks with health checks to verify deployments succeeded, and use multi-environment webhooks to promote changes through Dev → Staging → Production in a controlled pipeline.
