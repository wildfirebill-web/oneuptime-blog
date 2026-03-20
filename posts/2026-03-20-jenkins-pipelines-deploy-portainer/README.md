# How to Use Jenkins Pipelines to Deploy to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Jenkins, CI/CD, Pipelines, Docker, Deployment

Description: Learn how to build Jenkins pipelines that build Docker images and deploy them to Portainer stacks automatically.

---

Jenkins can deploy to Portainer by calling the Portainer API or stack webhooks at the end of a pipeline. This guide covers deploying Jenkins itself via Portainer and building deployment pipelines.

## Deploying Jenkins via Portainer

Start Jenkins as a Portainer stack:

```yaml
version: "3.8"

services:
  jenkins:
    image: jenkins/jenkins:lts
    user: root
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      JAVA_OPTS: "-Djenkins.install.runSetupWizard=false"
    networks:
      - jenkins_net

volumes:
  jenkins_home:

networks:
  jenkins_net:
    driver: bridge
```

Mounting `docker.sock` allows Jenkins to build images directly on the host.

## Declarative Jenkinsfile

A complete pipeline that builds, pushes, and deploys via Portainer:

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME    = 'myregistry.example.com/my-app'
        IMAGE_TAG     = "${env.GIT_COMMIT[0..7]}"
        PORTAINER_URL = 'https://portainer.example.com'
        STACK_NAME    = 'my-app'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest"
            }
        }

        stage('Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'registry-credentials',
                    usernameVariable: 'REGISTRY_USER',
                    passwordVariable: 'REGISTRY_PASS'
                )]) {
                    sh "echo $REGISTRY_PASS | docker login myregistry.example.com -u $REGISTRY_USER --password-stdin"
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'portainer-credentials',
                    usernameVariable: 'PORTAINER_USER',
                    passwordVariable: 'PORTAINER_PASS'
                )]) {
                    script {
                        def token = sh(
                            script: """
                                curl -s -X POST ${PORTAINER_URL}/api/auth \
                                  -H 'Content-Type: application/json' \
                                  -d '{"Username":"${PORTAINER_USER}","Password":"${PORTAINER_PASS}"}' \
                                  | jq -r .jwt
                            """,
                            returnStdout: true
                        ).trim()

                        def stackId = sh(
                            script: """
                                curl -s -H "Authorization: Bearer ${token}" \
                                  ${PORTAINER_URL}/api/stacks | \
                                  jq -r '.[] | select(.Name=="${STACK_NAME}-staging") | .Id'
                            """,
                            returnStdout: true
                        ).trim()

                        sh """
                            curl -s -X POST \
                              -H "Authorization: Bearer ${token}" \
                              "${PORTAINER_URL}/api/stacks/${stackId}/images/update?pullImage=true"
                        """
                    }
                }
            }
        }

        stage('Smoke Test') {
            steps {
                sh './scripts/smoke-test.sh https://staging.example.com'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                sh "curl -X POST ${env.PORTAINER_PROD_WEBHOOK}"
            }
        }
    }

    post {
        failure {
            mail to: 'team@example.com',
                 subject: "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "See ${env.BUILD_URL} for details."
        }
    }
}
```

## Storing Credentials Securely in Jenkins

Never hardcode Portainer credentials in Jenkinsfiles. Use Jenkins Credential Store:

1. Go to **Manage Jenkins > Credentials > Global**.
2. Add a **Username with password** credential with ID `portainer-credentials`.
3. Reference it with `withCredentials` as shown in the pipeline above.

## Parallel Deployments

Deploy multiple services simultaneously to reduce pipeline duration:

```groovy
stage('Deploy Services') {
    parallel {
        stage('Deploy API') {
            steps {
                sh "curl -X POST ${PORTAINER_WEBHOOK_API}"
            }
        }
        stage('Deploy Worker') {
            steps {
                sh "curl -X POST ${PORTAINER_WEBHOOK_WORKER}"
            }
        }
        stage('Deploy Frontend') {
            steps {
                sh "curl -X POST ${PORTAINER_WEBHOOK_FRONTEND}"
            }
        }
    }
}
```
