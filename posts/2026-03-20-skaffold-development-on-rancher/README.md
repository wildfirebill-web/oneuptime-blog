# How to Set Up Skaffold for Development on Rancher - Development

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Skaffold, Developer Experience, Kubernetes, CI/CD, Local Development

Description: Configure Skaffold for iterative application development on a Rancher-managed Kubernetes cluster with automatic build, push, and deploy on code changes.

## Introduction

Skaffold is a command-line tool that handles the development workflow for Kubernetes applications: it watches your source code, rebuilds the container image on change, pushes it to a registry, and deploys the updated image to your cluster. This enables a fast inner development loop on real Kubernetes infrastructure.

## Prerequisites

- Rancher cluster accessible via `kubectl`
- Docker or Buildpacks for building images
- A container registry (Docker Hub, ECR, GCR, or local registry)

## Step 1: Install Skaffold

```bash
# macOS

brew install skaffold

# Linux
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
chmod +x skaffold && sudo mv skaffold /usr/local/bin
```

## Step 2: Create a Skaffold Configuration

```yaml
# skaffold.yaml
apiVersion: skaffold/v4beta6
kind: Config
metadata:
  name: myapp

build:
  artifacts:
    - image: myregistry/myapp         # Your container registry path
      context: .                       # Build context (current directory)
      docker:
        dockerfile: Dockerfile
  local:
    push: false   # Set to true for remote clusters, false for local

tagPolicy:
  sha256: {}    # Tag images with their content hash for immutability

deploy:
  kubectl:
    manifests:
      - k8s/deployment.yaml
      - k8s/service.yaml
      - k8s/ingress.yaml

portForward:
  - resourceType: service
    resourceName: myapp
    port: 8080
    localPort: 8080    # Access http://localhost:8080 during development
```

## Step 3: Configure Kubernetes Manifests with Image Templates

Skaffold automatically substitutes the correct image tag in your manifests:

```yaml
# k8s/deployment.yaml
spec:
  containers:
    - name: myapp
      image: myregistry/myapp    # Skaffold replaces this with the built tag
```

## Step 4: Run Development Mode

```bash
# Start Skaffold in development mode
# It watches files, rebuilds on change, and deploys automatically
skaffold dev --kubeconfig ~/.kube/rancher-production.yaml

# Skaffold output shows:
# - Build progress
# - Deploy status
# - Container logs in real time
# - Port-forward status
```

## Step 5: Run Once for CI/CD

```bash
# Build and deploy once (for CI pipelines)
skaffold run \
  --default-repo=myregistry \
  --kubeconfig ~/.kube/rancher-production.yaml

# Build image only (no deploy)
skaffold build --tag=v1.2.3
```

## Step 6: Configure Profiles for Different Environments

```yaml
# skaffold.yaml profiles section
profiles:
  - name: production
    deploy:
      helm:
        releases:
          - name: myapp
            chartPath: charts/myapp
            setValues:
              replicaCount: 3
              resources.requests.memory: 512Mi

  - name: development
    deploy:
      kubectl:
        manifests:
          - k8s/dev/deployment.yaml    # Lighter dev config
```

```bash
# Use production profile
skaffold run --profile production
```

## Conclusion

Skaffold eliminates the manual rebuild-retag-push-deploy cycle during Kubernetes development. In `dev` mode, code changes trigger automatic redeployment in seconds, providing a development experience approaching local development while testing against real cluster infrastructure.
