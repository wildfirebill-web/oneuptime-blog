# How to Deploy Jenkins via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Jenkins, CI/CD, DevOps, Self-Hosted

Description: Deploy Jenkins via Portainer with persistent storage, Docker-in-Docker support for building images, and pre-installed plugins for a ready-to-use CI/CD platform.

## Introduction

Jenkins is the most widely deployed open-source CI/CD server. Deploying it via Portainer with the Docker socket mounted allows Jenkins to build and push Docker images as part of CI pipelines, while persistent storage ensures your job history and configurations survive updates.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    container_name: jenkins
    user: root   # Required for Docker socket access
    environment:
      - JENKINS_OPTS=--httpPort=8080
      - JAVA_OPTS=-Xmx2g -Xms512m
    volumes:
      # Persistent Jenkins home directory
      - jenkins_home:/var/jenkins_home
      # Docker socket for building images in pipelines
      - /var/run/docker.sock:/var/run/docker.sock
      # Docker CLI for pipeline use
      - /usr/bin/docker:/usr/bin/docker:ro
    ports:
      - "8080:8080"    # Jenkins web UI
      - "50000:50000"  # Jenkins agent port
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:8080/login | grep -q 'Jenkins'"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

volumes:
  jenkins_home:
```

## Initial Setup

```bash
# Get initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Navigate to `http://<host>:8080`, enter the password, and install suggested plugins.

## Installing Plugins via CLI

```bash
# Install plugins using Jenkins CLI
docker exec jenkins jenkins-plugin-cli --plugins \
  git \
  docker-workflow \
  pipeline-stage-view \
  blueocean \
  credentials-binding \
  kubernetes \
  prometheus
```

## Example Declarative Pipeline

Create a Jenkinsfile in your repository:

```groovy
// Jenkinsfile - Docker build and push pipeline
pipeline {
    agent any
    
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE_NAME = 'myapp'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image
                    dockerImage = docker.build("${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    // Run tests inside the container
                    dockerImage.inside {
                        sh 'npm test'
                    }
                }
            }
        }
        
        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'registry-credentials') {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy via Portainer') {
            steps {
                script {
                    // Trigger Portainer webhook to update stack
                    sh """
                        curl -X POST \
                          'https://portainer.example.com/api/stacks/webhook/YOUR_WEBHOOK_ID' \
                          -H 'Content-Type: application/json'
                    """
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline succeeded! Image: ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
```

## Configure Jenkins Credentials

In Jenkins UI:

1. Navigate to **Manage Jenkins > Credentials**
2. Add credentials for:
   - Docker registry (username/password)
   - Git repository (SSH key or token)
   - Portainer API token

## Jenkins + Portainer Integration

Generate a Portainer webhook for automated deployments:

1. In Portainer, navigate to **Stacks**
2. Click on your application stack
3. Enable **GitOps updates**
4. Copy the webhook URL
5. Use the webhook URL in your Jenkins pipeline to trigger updates

## Conclusion

Jenkins deployed via Portainer provides a powerful CI/CD platform with Docker build capabilities. The Docker socket mount allows pipelines to build, test, and push images directly. Combined with Portainer's webhook feature, you can build complete GitOps pipelines that automatically update deployments when new images are pushed.
