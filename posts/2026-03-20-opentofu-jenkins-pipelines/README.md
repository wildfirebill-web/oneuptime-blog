# How to Set Up OpenTofu with Jenkins Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Jenkins, CI/CD, Infrastructure as Code, Automation

Description: Learn how to integrate OpenTofu into Jenkins pipelines with plan-on-PR and apply-on-merge workflows, credential management, and Slack notifications.

## Introduction

Jenkins is a widely used CI/CD platform that integrates well with OpenTofu. This guide builds a Jenkinsfile that runs `tofu plan` on pull requests and `tofu apply` on merges to main, with AWS credentials sourced from Jenkins Credentials.

## Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        TF_VERSION = "1.7.0"
        TF_DIR     = "environments/production"
    }

    stages {
        stage('Setup OpenTofu') {
            steps {
                sh """
                    wget -q https://github.com/opentofu/opentofu/releases/download/v${TF_VERSION}/tofu_${TF_VERSION}_linux_amd64.zip
                    unzip -o tofu_${TF_VERSION}_linux_amd64.zip -d /usr/local/bin
                    chmod +x /usr/local/bin/tofu
                    tofu version
                """
            }
        }

        stage('Init') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh "tofu -chdir=${TF_DIR} init -input=false"
                }
            }
        }

        stage('Plan') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        tofu -chdir=${TF_DIR} plan \
                            -out=tfplan \
                            -input=false \
                            -no-color \
                            2>&1 | tee plan-output.txt
                    """
                }
                archiveArtifacts artifacts: 'plan-output.txt', fingerprint: true
            }
        }

        stage('Apply') {
            when {
                branch 'main'
            }
            steps {
                // Require manual approval before applying
                input message: 'Apply the Terraform plan?', ok: 'Apply'

                withCredentials([
                    string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh "tofu -chdir=${TF_DIR} apply -input=false -auto-approve tfplan"
                }
            }
        }
    }

    post {
        failure {
            slackSend(
                color: 'danger',
                message: "OpenTofu pipeline failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
        }
        success {
            slackSend(
                color: 'good',
                message: "OpenTofu pipeline succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
        }
    }
}
```

## Caching Providers

Speed up pipelines by caching the `.terraform` directory between builds:

```groovy
stage('Init') {
    steps {
        cache(maxCacheSize: 500, caches: [
            arbitraryFileCache(path: '.terraform', cacheValidityDecidingFile: '.terraform.lock.hcl')
        ]) {
            sh "tofu init -input=false"
        }
    }
}
```

## Conclusion

Jenkins pipelines with OpenTofu work best when credentials are stored in the Jenkins Credentials store (never in the Jenkinsfile), plans are archived as build artefacts, and manual approval gates protect the apply stage in production.
