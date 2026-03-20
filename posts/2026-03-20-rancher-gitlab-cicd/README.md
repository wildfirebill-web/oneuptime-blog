# How to Integrate GitLab CI/CD with Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, GitLab, CI/CD

Description: Integrate GitLab CI/CD with Rancher to build container images, run tests on Kubernetes runners, and deploy applications to Rancher-managed clusters.

## Introduction

GitLab CI/CD and Rancher complement each other naturally: GitLab manages source code and pipelines, while Rancher manages Kubernetes clusters. This integration enables pipelines that build Docker images, run tests using Kubernetes-based GitLab Runners, and deploy directly to Rancher-managed clusters using kubeconfig-based authentication.

## Prerequisites

- GitLab instance (self-hosted or GitLab.com)
- Rancher with at least one downstream cluster
- Docker registry (GitLab Container Registry or external)

## Step 1: Install GitLab Runner on a Rancher Cluster

```bash
# Add the GitLab Helm repository

helm repo add gitlab https://charts.gitlab.io
helm repo update

# Retrieve your GitLab Runner registration token
# GitLab UI: Project/Group → Settings → CI/CD → Runners → Registration Token

# Install GitLab Runner
helm install gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab-runner \
  --create-namespace \
  --set gitlabUrl=https://gitlab.example.com \
  --set runnerRegistrationToken=<registration-token> \
  --set rbac.create=true \
  --set runners.executor=kubernetes \
  --set runners.kubernetes.namespace=gitlab-runner \
  --set runners.privileged=true    # Required for Docker-in-Docker builds
```

## Step 2: Configure CI/CD Variables in GitLab

Store the Rancher kubeconfig as a CI/CD variable:

```bash
# Get kubeconfig from Rancher (for the target cluster)
curl -sk -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "https://rancher.example.com/v3/clusters/<cluster-id>?action=generateKubeconfig" \
  | jq -r .config | base64 -w 0
# → Copy the base64 output
```

In GitLab UI:
1. **Project → Settings → CI/CD → Variables**.
2. Add variable:
   - **Key**: `KUBECONFIG_B64`
   - **Value**: (paste base64 kubeconfig)
   - **Type**: Variable
   - Check **Mask variable**

## Step 3: Basic CI/CD Pipeline (.gitlab-ci.yml)

```yaml
# .gitlab-ci.yml

stages:
  - build
  - test
  - deploy

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  KUBECTL_VERSION: "1.29"

# Build the Docker image
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
  only:
    - main
    - merge_requests

# Run unit tests
test:
  stage: test
  image: maven:3.9-jdk-17
  script:
    - mvn test
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml
  only:
    - main
    - merge_requests

# Deploy to Rancher-managed cluster
deploy-staging:
  stage: deploy
  image: bitnami/kubectl:${KUBECTL_VERSION}
  script:
    # Decode the kubeconfig
    - echo "$KUBECONFIG_B64" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig

    # Update the deployment
    - kubectl set image deployment/myapp myapp=$IMAGE_TAG -n staging
    - kubectl rollout status deployment/myapp -n staging --timeout=5m
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - main

# Deploy to production (manual gate)
deploy-production:
  stage: deploy
  image: bitnami/kubectl:${KUBECTL_VERSION}
  script:
    - echo "$KUBECONFIG_PROD_B64" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
    - kubectl set image deployment/myapp myapp=$IMAGE_TAG -n production
    - kubectl rollout status deployment/myapp -n production --timeout=5m
  environment:
    name: production
    url: https://myapp.example.com
  when: manual    # Require manual approval
  only:
    - main
```

## Step 4: Use Helm for Deployments

```yaml
# Deploy with Helm in CI/CD
deploy-helm:
  stage: deploy
  image:
    name: alpine/helm:3.14
    entrypoint: [""]
  script:
    - echo "$KUBECONFIG_B64" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig

    # Upgrade or install the Helm release
    - helm upgrade --install myapp ./charts/myapp \
        --namespace production \
        --create-namespace \
        --set image.tag=$CI_COMMIT_SHORT_SHA \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --atomic \             # Roll back automatically on failure
        --timeout 5m
```

## Step 5: Multi-Cluster Deployment

```yaml
# Deploy to multiple Rancher clusters using a matrix
.deploy-template: &deploy-template
  stage: deploy
  image: bitnami/kubectl:1.29
  script:
    - echo "$KUBECONFIG" | base64 -d > /tmp/kubeconfig
    - kubectl set image deployment/myapp myapp=$IMAGE_TAG -n production --kubeconfig=/tmp/kubeconfig
    - kubectl rollout status deployment/myapp -n production --kubeconfig=/tmp/kubeconfig

deploy-us:
  <<: *deploy-template
  variables:
    KUBECONFIG: $KUBECONFIG_US_B64
  environment:
    name: production-us

deploy-eu:
  <<: *deploy-template
  variables:
    KUBECONFIG: $KUBECONFIG_EU_B64
  environment:
    name: production-eu
```

## Step 6: Notify Rancher on Deployment

```yaml
# Update Rancher with deployment status via API
notify-rancher:
  stage: deploy
  image: curlimages/curl:latest
  script:
    - |
      curl -sk -X POST \
        -H "Authorization: Bearer $RANCHER_API_TOKEN" \
        -H "Content-Type: application/json" \
        -d "{\"description\": \"Deployed ${CI_COMMIT_SHORT_SHA} by ${GITLAB_USER_NAME}\"}" \
        "https://rancher.example.com/v3/clusters/<cluster-id>/clusterregistrationtokens"
```

## Conclusion

GitLab CI/CD with Rancher provides a complete DevSecOps pipeline: code committed to GitLab triggers automated builds, tests, and deployments to Rancher-managed clusters. Kubernetes-native GitLab Runners eliminate the need for static build infrastructure, and manual gates for production deployments provide a safety checkpoint. This integration supports multi-cluster deployments and provides full audit trails in both GitLab and Rancher.
