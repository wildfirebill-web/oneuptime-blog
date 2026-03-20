# How to Configure DevSpace with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, DevSpace, Developer Experience, Kubernetes, Local Development, Helm

Description: Set up DevSpace for collaborative Kubernetes development in Rancher with shared dev clusters, namespace isolation, and automatic synchronization.

## Introduction

DevSpace is a developer tool for Kubernetes that provides hot reloading, file sync, and multi-service development workflows. Unlike Skaffold and Tilt, DevSpace has built-in support for team workflows, namespace isolation per developer, and deep Helm chart integration.

## Step 1: Install DevSpace

```bash
# macOS

brew install devspace

# Linux/macOS via curl
curl -L -o devspace \
  "https://github.com/loft-sh/devspace/releases/latest/download/devspace-linux-amd64"
chmod +x devspace && sudo mv devspace /usr/local/bin
```

## Step 2: Initialize DevSpace in Your Project

```bash
cd my-application
devspace init
# Follow the interactive setup wizard
```

Or create the configuration manually:

```yaml
# devspace.yaml
version: v2beta1
name: myapp

images:
  myapp:
    image: myregistry/myapp
    dockerfile: ./Dockerfile
    buildArgs:
      ENV: development

deployments:
  myapp:
    helm:
      chart:
        name: ./charts/myapp    # Local Helm chart
      values:
        image:
          repository: myregistry/myapp
          tag: latest
        replicaCount: 1
        resources:
          requests:
            memory: "128Mi"
            cpu: "50m"

dev:
  myapp:
    imageSelector: myregistry/myapp   # Attach to the pod running this image
    command: ["npm", "run", "dev"]    # Override container command
    sync:
      - path: ./src:/app/src          # Sync local src to container
        excludePaths:
          - node_modules
          - .git
    ports:
      - port: "8080"                  # Forward container port to local
    logs:
      enabled: true                   # Stream logs to terminal
```

## Step 3: Configure Namespace Isolation (Team Workfow)

DevSpace supports per-developer namespaces for isolation:

```yaml
# devspace.yaml additions
vars:
  DEVELOPER_NAME:
    default: ""    # Set via --var flag

pipelines:
  dev:
    run: |-
      create_deployments myapp
      start_dev myapp

localRegistry:
  enabled: false    # Use external registry for shared clusters
```

```bash
# Each developer deploys to their own namespace
devspace dev --var DEVELOPER_NAME=alice --namespace dev-alice
devspace dev --var DEVELOPER_NAME=bob   --namespace dev-bob
```

## Step 4: Start Development Mode

```bash
# Start devspace dev against your Rancher cluster
devspace dev --kube-context rancher-dev-cluster --namespace myapp-dev

# DevSpace will:
# 1. Build and push the container image
# 2. Deploy the Helm chart
# 3. Start file synchronization
# 4. Stream container logs
```

## Step 5: Deploy to Staging/Production

```bash
# Deploy to staging (builds optimized image, no dev overrides)
devspace deploy --profile staging --kube-context rancher-staging

# Deploy to production
devspace deploy --profile production --kube-context rancher-production
```

## Conclusion

DevSpace on Rancher provides a production-grade inner development loop with built-in team workflows. Namespace isolation prevents developers from interfering with each other, while file sync and hot reloading minimize iteration time. The Helm integration means development configuration stays consistent with production deployments.
