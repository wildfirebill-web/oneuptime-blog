# How to Set Up Jenkins Pipelines That Deploy to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Jenkins, CI/CD, DevOps, Automation, Pipeline

Description: Configure Jenkins declarative pipelines to build Docker images and deploy stacks to Portainer using the Portainer API and webhooks.

## Introduction

Jenkins is the most widely deployed CI/CD automation server. This guide covers deploying Jenkins alongside Portainer, configuring pipelines that build Docker images, run tests, and deploy to Portainer environments - all from a single Jenkinsfile.

## Step 1: Deploy Jenkins with Docker Support

```yaml
# docker-compose.yml - Jenkins for Docker builds

version: "3.8"

networks:
  jenkins_network:
    driver: bridge

volumes:
  jenkins_home:
  jenkins_agents:

services:
  jenkins:
    image: jenkins/jenkins:lts-jdk21
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    environment:
      - JAVA_OPTS=-Xmx2g -Xms512m
      - JENKINS_OPTS=--httpPort=8080
    volumes:
      - jenkins_home:/var/jenkins_home
      # Docker access for builds
      - /var/run/docker.sock:/var/run/docker.sock
    user: root
    networks:
      - jenkins_network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jenkins.rule=Host(`jenkins.yourdomain.com`)"
      - "traefik.http.routers.jenkins.entrypoints=websecure"
      - "traefik.http.services.jenkins.loadbalancer.server.port=8080"
```

## Step 2: Install Jenkins Docker Tools

```groovy
// bootstrap.groovy - Install plugins on first run
// Place in /var/jenkins_home/init.groovy.d/

import jenkins.model.Jenkins

def plugins = [
    "docker-plugin",
    "docker-workflow",
    "git",
    "github",
    "credentials-binding",
    "pipeline-utility-steps",
    "blueocean",
    "email-ext",
    "slack"
]

def pm = Jenkins.instance.pluginManager
def uc = Jenkins.instance.updateCenter

plugins.each { plugin ->
    if (!pm.getPlugin(plugin)) {
        uc.getPlugin(plugin).deploy()
    }
}
```

## Step 3: Configure Credentials in Jenkins

1. Navigate to **Manage Jenkins** > **Credentials** > **System** > **Global credentials**
2. Add the following:

| ID | Type | Value |
|----|------|-------|
| `portainer-api-key` | Secret text | Your Portainer API key |
| `docker-registry` | Username/password | Registry credentials |
| `portainer-url` | Secret text | `https://portainer.yourdomain.com` |

Generate Portainer API key:
1. Portainer > Account > Access Tokens > Add access token
2. Copy the generated token

## Step 4: Complete Declarative Pipeline

```groovy
// Jenkinsfile - Comprehensive deployment pipeline
pipeline {
    agent {
        docker {
            image 'docker:24-dind'
            args '--privileged -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['staging', 'production'],
            description: 'Target deployment environment'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip test execution'
        )
    }

    environment {
        PORTAINER_URL  = credentials('portainer-url')
        PORTAINER_KEY  = credentials('portainer-api-key')
        REGISTRY       = "registry.yourdomain.com"
        APP_NAME       = "myapp"
        IMAGE          = "${REGISTRY}/${APP_NAME}"
        // Environment-specific stack IDs
        STAGING_STACK_ID    = "1"
        PRODUCTION_STACK_ID = "2"
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    // Compute image tag from git info
                    env.GIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    env.IMAGE_TAG = "${BUILD_NUMBER}-${env.GIT_SHORT}"
                    env.FULL_IMAGE = "${IMAGE}:${env.IMAGE_TAG}"

                    echo "Building: ${env.FULL_IMAGE}"
                }
            }
        }

        stage('Test') {
            when {
                not { expression { params.SKIP_TESTS } }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh '''
                            docker run --rm \
                                -v "$PWD:/app" \
                                -w /app \
                                python:3.12-slim \
                                sh -c "
                                    pip install -q -r requirements.txt &&
                                    pip install -q pytest pytest-cov &&
                                    pytest tests/unit/ -v --junitxml=test-results/unit.xml
                                "
                        '''
                        junit 'test-results/*.xml'
                    }
                }

                stage('Linting') {
                    steps {
                        sh '''
                            docker run --rm \
                                -v "$PWD:/app" \
                                -w /app \
                                python:3.12-slim \
                                sh -c "pip install -q flake8 && flake8 src/"
                        '''
                    }
                }
            }
        }

        stage('Build') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-registry',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        docker login ${REGISTRY} -u ${DOCKER_USER} -p ${DOCKER_PASS}

                        # Build with cache from latest
                        docker build \
                            --cache-from ${IMAGE}:latest \
                            --build-arg BUILD_NUMBER=${BUILD_NUMBER} \
                            --build-arg GIT_COMMIT=${GIT_SHORT} \
                            -t ${FULL_IMAGE} \
                            -t ${IMAGE}:latest \
                            .

                        docker push ${FULL_IMAGE}
                        docker push ${IMAGE}:latest

                        echo "Pushed: ${FULL_IMAGE}"
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                anyOf {
                    branch 'develop'
                    expression { params.ENVIRONMENT == 'staging' }
                }
            }
            steps {
                script {
                    deployToPortainer(
                        stackId: STAGING_STACK_ID,
                        imageTag: env.IMAGE_TAG
                    )
                }
            }
        }

        stage('Integration Tests') {
            when {
                branch 'develop'
            }
            steps {
                sh """
                    # Wait for staging to be healthy
                    sleep 30

                    # Run integration tests against staging
                    docker run --rm \
                        -e TARGET_URL=https://staging.yourdomain.com \
                        ${IMAGE}:${IMAGE_TAG} \
                        python -m pytest tests/integration/ -v
                """
            }
        }

        stage('Deploy to Production') {
            when {
                anyOf {
                    branch 'main'
                    expression { params.ENVIRONMENT == 'production' }
                }
            }
            steps {
                input(
                    message: "Deploy ${IMAGE_TAG} to PRODUCTION?",
                    ok: "Deploy"
                )
                script {
                    deployToPortainer(
                        stackId: PRODUCTION_STACK_ID,
                        imageTag: env.IMAGE_TAG
                    )
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout ${REGISTRY} || true'
            cleanWs()
        }
        success {
            echo "Successfully deployed ${env.IMAGE_TAG}"
        }
        failure {
            emailext(
                subject: "FAILED: ${JOB_NAME} #${BUILD_NUMBER}",
                body: "Build failed. Details: ${BUILD_URL}",
                to: "devops@yourdomain.com"
            )
        }
    }
}

// Helper function to deploy via Portainer API
def deployToPortainer(Map config) {
    sh """
        # Update stack image tag via Portainer API
        curl -s -X PUT \
            -H "X-API-Key: ${PORTAINER_KEY}" \
            -H "Content-Type: application/json" \
            "${PORTAINER_URL}/api/stacks/${config.stackId}?endpointId=1" \
            -d '{
                "stackFileContent": "$(cat docker-compose.yml | python3 -c "import sys,json; print(json.dumps(sys.stdin.read()))")",
                "env": [
                    {"name": "IMAGE_TAG", "value": "${config.imageTag}"}
                ],
                "prune": false
            }' | python3 -c "
import json,sys
r = json.load(sys.stdin)
if 'Id' in r:
    print('Stack updated successfully: ID=' + str(r['Id']))
else:
    print('Error: ' + json.dumps(r))
    sys.exit(1)
"
    """
}
```

## Step 5: Multibranch Pipeline Configuration

```groovy
// Jenkins Job DSL - Create multibranch pipeline automatically
multibranchPipelineJob('myapp-pipeline') {
    branchSources {
        gitHub {
            id('myapp-github')
            repoOwner('yourorg')
            repository('myapp')
            credentialsId('github-token')
        }
    }
    orphanedItemStrategy {
        discardOldItems {
            numToKeep(10)
        }
    }
    triggers {
        periodic(5)  // Poll every 5 minutes
    }
}
```

## Conclusion

Jenkins pipelines with Portainer deployments give you a powerful, self-hosted CI/CD system. The declarative pipeline syntax makes it readable and maintainable, parallel stages speed up the pipeline, and the Portainer API integration ensures deployments are atomic and trackable. Use Jenkins shared libraries to extract common pipeline steps (like `deployToPortainer`) into a reusable library across all your projects.
