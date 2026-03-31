# How to Set Up Service Webhooks in Portainer on Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Webhook, CI/CD, DevOps

Description: Learn how to configure Portainer service webhooks on Docker Swarm to enable automated image updates triggered by CI/CD pipelines.

## Introduction

Portainer service webhooks provide an HTTP endpoint that, when called, triggers an image update for a Swarm service. This enables a push-based deployment model where your CI/CD pipeline builds a new image and then tells Portainer to update the service - without needing to store Portainer credentials in your CI system. This guide covers setting up and using service webhooks.

## Prerequisites

- Portainer CE or BE on Docker Swarm
- A Swarm service to update
- A CI/CD system (GitHub Actions, GitLab CI, Jenkins, etc.)

## Step 1: Enable the Service Webhook

1. Navigate to **Services** in Portainer
2. Click on the service you want to configure
3. Click **Edit this service** or find the **Webhooks** section
4. Enable **Service webhook**
5. Copy the generated webhook URL

The URL looks like:

```text
https://portainer.example.com:9443/api/webhooks/abc123def456...
```

## Step 2: Test the Webhook

Test the webhook with curl to verify it works:

```bash
# Trigger a service update via webhook

curl -X POST \
  "https://portainer.example.com:9443/api/webhooks/abc123def456" \
  --insecure

# Expected: HTTP 200 OK
# Portainer will pull the latest image and update the service
```

## Step 3: Integrate with GitHub Actions

Add the webhook call to your CI/CD pipeline:

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            myorg/myapp:latest
            myorg/myapp:${{ github.sha }}

      - name: Trigger Portainer service update
        run: |
          curl -X POST \
            "${{ secrets.PORTAINER_WEBHOOK_URL }}" \
            --fail \
            --max-time 30
```

Store the webhook URL as a GitHub secret (`PORTAINER_WEBHOOK_URL`).

## Step 4: Integrate with GitLab CI

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
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

deploy:
  stage: deploy
  image: alpine:latest
  script:
    - apk add --no-cache curl
    - |
      curl -X POST "$PORTAINER_WEBHOOK_URL" \
        --fail \
        --max-time 30
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
                sh '''
                    docker build -t myapp:${BUILD_NUMBER} .
                    docker tag myapp:${BUILD_NUMBER} myapp:latest
                    docker push myapp:latest
                '''
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([string(credentialsId: 'portainer-webhook', variable: 'WEBHOOK_URL')]) {
                    sh '''
                        curl -X POST "${WEBHOOK_URL}" \
                          --fail \
                          --max-time 30
                    '''
                }
            }
        }
    }
}
```

## Step 6: Understand the Webhook Behavior

When the webhook is called:

1. Portainer receives the POST request
2. For each task in the service, Portainer pulls the **latest** version of the configured image tag
3. If the digest has changed, Portainer updates the service
4. If the digest is the same (image unchanged), behavior depends on the **Force update** setting

### Force Update

To always recreate tasks even if the image digest is unchanged:

In the service webhook settings, enable **Force update**. This is useful when using the `latest` tag, where you want to force a re-pull and re-create even if the tag points to the same digest.

## Step 7: Secure the Webhook

The webhook URL is a secret - anyone with the URL can trigger a deployment. Protect it:

1. **Store as a CI/CD secret** - Never hardcode in your pipeline files
2. **Restrict network access** - Use firewall rules to limit which IPs can reach Portainer's API
3. **Rotate webhooks** - Regenerate the webhook URL periodically or after team members leave
4. **Use HTTPS only** - Never expose Portainer over plain HTTP

## Monitoring Webhook-Triggered Deployments

After a webhook triggers a deployment, monitor in Portainer:

1. Go to **Services** and find the service
2. Check the **Tasks** list for new tasks starting
3. Verify tasks reach **Running** state
4. Check **Logs** for any startup errors

## Conclusion

Portainer service webhooks bridge the gap between your CI/CD pipelines and your Swarm cluster. By triggering image updates via webhook, you achieve automated continuous deployment without sharing Portainer credentials with your build systems. This approach scales well across multiple services and can easily integrate with any CI/CD platform that can make HTTP requests.
