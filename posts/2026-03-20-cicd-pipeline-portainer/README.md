# How to Deploy a Complete CI/CD Pipeline with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, CI/CD, DevOps, Jenkins, Gitea, Pipelines

Description: Build a complete self-hosted CI/CD pipeline with Gitea, Jenkins, and Portainer webhooks for automated build, test, and deployment workflows.

## Introduction

A complete CI/CD pipeline automates the journey from code commit to production deployment. This guide builds an entirely self-hosted pipeline using Gitea (Git hosting), Jenkins (CI/CD), a Docker registry, and Portainer webhooks for deployments - no reliance on GitHub Actions or external services.

## Architecture

```text
Developer → Gitea (Git) → Jenkins (Build/Test) → Registry → Portainer (Deploy)
```

## Step 1: Deploy the CI/CD Infrastructure Stack

```yaml
# docker-compose.yml - Self-hosted CI/CD Stack

version: "3.8"

networks:
  cicd_network:
    driver: bridge

volumes:
  gitea_data:
  gitea_db:
  jenkins_home:
  registry_data:

services:
  # Gitea - Self-hosted Git
  gitea_db:
    image: postgres:15-alpine
    container_name: gitea_db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=gitea
      - POSTGRES_USER=gitea
      - POSTGRES_PASSWORD=gitea_db_password
    volumes:
      - gitea_db:/var/lib/postgresql/data
    networks:
      - cicd_network

  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: unless-stopped
    ports:
      - "3000:3000"
      - "222:22"    # Git SSH
    environment:
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=gitea_db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=gitea_db_password
      - GITEA__server__DOMAIN=git.yourdomain.com
      - GITEA__server__ROOT_URL=https://git.yourdomain.com/
      - GITEA__server__SSH_DOMAIN=git.yourdomain.com
      - GITEA__service__DISABLE_REGISTRATION=false
    volumes:
      - gitea_data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    networks:
      - cicd_network
    depends_on:
      - gitea_db

  # Jenkins CI/CD
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"    # Jenkins agents
    environment:
      - JENKINS_OPTS=--prefix=/jenkins
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock  # For Docker builds
    networks:
      - cicd_network
    user: root  # Needed for Docker access

  # Private Docker Registry
  registry:
    image: registry:2
    container_name: docker_registry
    restart: unless-stopped
    ports:
      - "5000:5000"
    environment:
      - REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/data
      - REGISTRY_HTTP_SECRET=registry_http_secret
    volumes:
      - registry_data:/data
    networks:
      - cicd_network

  # Registry UI
  registry_ui:
    image: joxit/docker-registry-ui:latest
    container_name: registry_ui
    restart: unless-stopped
    ports:
      - "8888:80"
    environment:
      - REGISTRY_URL=http://registry:5000
      - REGISTRY_TITLE=My Docker Registry
    networks:
      - cicd_network
    depends_on:
      - registry
```

## Step 2: Configure Jenkins Pipeline

Install Jenkins plugins:
- Docker Pipeline
- Git
- Credentials
- Blue Ocean (optional)

```groovy
// Jenkinsfile - Complete CI/CD pipeline
pipeline {
    agent any

    environment {
        // Docker registry
        REGISTRY = "registry:5000"
        IMAGE_NAME = "${REGISTRY}/myapp/api"
        IMAGE_TAG = "${BUILD_NUMBER}-${GIT_COMMIT.take(8)}"

        // Portainer webhook (from Portainer Stack settings)
        PORTAINER_WEBHOOK = credentials('portainer-webhook-url')
        PORTAINER_API_KEY = credentials('portainer-api-key')
        PORTAINER_URL = "https://portainer.yourdomain.com"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh '''
                    docker run --rm \
                        -v "$PWD":/app \
                        -w /app \
                        python:3.12-slim \
                        sh -c "pip install -q -r requirements-test.txt && pytest tests/ -v"
                '''
            }
        }

        stage('Build') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}", "--no-cache .")
                    docker.build("${IMAGE_NAME}:latest", ".")
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    docker.withRegistry("http://${REGISTRY}") {
                        docker.image("${IMAGE_NAME}:${IMAGE_TAG}").push()
                        docker.image("${IMAGE_NAME}:latest").push()
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh """
                    curl -X POST \
                        -H "X-API-Key: ${PORTAINER_API_KEY}" \
                        "${PORTAINER_URL}/api/stacks/webhooks/${PORTAINER_WEBHOOK}"
                """
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
            }
            steps {
                sh """
                    curl -X POST \
                        -H "X-API-Key: ${PORTAINER_API_KEY}" \
                        "${PORTAINER_URL}/api/stacks/1/git/redeploy" \
                        -H "Content-Type: application/json" \
                        -d '{"pullImage": true, "prune": false}'
                """
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully: ${IMAGE_TAG}"
        }
        failure {
            // Send notification
            mail(
                to: 'dev@yourdomain.com',
                subject: "Pipeline FAILED: ${JOB_NAME} #${BUILD_NUMBER}",
                body: "Build failed. Check: ${BUILD_URL}"
            )
        }
    }
}
```

## Step 3: Configure Gitea Webhook to Trigger Jenkins

1. In Gitea: **Settings** > **Webhooks** > **Add Webhook** > **Gitea**
2. Target URL: `http://jenkins:8080/gitea-webhook/post`
3. Secret: (same as Jenkins Gitea plugin config)
4. Trigger: Push events, Pull request events

## Step 4: Portainer Webhook Setup

1. In Portainer: Navigate to your stack
2. Click **Edit** > find the **Git polling** or **Webhook** section
3. Enable **GitOps updates**
4. Copy the webhook URL
5. Store it as a Jenkins credential

## Step 5: Multi-Environment Stack Updates via API

```bash
#!/bin/bash
# deploy-to-portainer.sh

PORTAINER_URL="$1"
API_KEY="$2"
STACK_ID="$3"
IMAGE_TAG="$4"

# Get current stack info
STACK=$(curl -s \
    -H "X-API-Key: $API_KEY" \
    "$PORTAINER_URL/api/stacks/$STACK_ID")

# Update image tag in environment variables
NEW_ENV=$(echo "$STACK" | jq --arg tag "$IMAGE_TAG" \
    '.Env | map(if .name == "IMAGE_TAG" then .value = $tag else . end)')

# Redeploy the stack
curl -X PUT \
    -H "X-API-Key: $API_KEY" \
    -H "Content-Type: application/json" \
    "$PORTAINER_URL/api/stacks/$STACK_ID?endpointId=1" \
    -d "{\"stackFileContent\": $(echo "$STACK" | jq -c '.'), \"env\": $NEW_ENV, \"prune\": false}"
```

## Conclusion

Your self-hosted CI/CD pipeline now automates the entire software delivery process. Gitea hosts your code, Jenkins builds, tests, and packages it, the private registry stores your Docker images, and Portainer webhooks trigger deployments. This stack runs entirely on your own infrastructure - no third-party dependencies, no data leaving your network, and no per-seat pricing. Portainer sits at the center of the deployment stage, making it easy to track what version is running in each environment.
