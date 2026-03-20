# How to Configure Drone CI with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Drone CI, CI/CD

Description: Set up Drone CI on a Rancher-managed Kubernetes cluster to run containerized pipelines and deploy applications to Rancher clusters.

## Introduction

Drone CI is a lightweight, container-native CI/CD platform that runs each pipeline step in a Docker container. Deploying Drone on a Rancher-managed cluster gives you Kubernetes-backed pipeline runners, persistent build history, and easy integration with popular source control systems. This guide covers deploying Drone on Rancher and configuring it to deploy to downstream clusters.

## Prerequisites

- Rancher managing a Kubernetes cluster
- GitHub or Gitea OAuth application credentials
- A domain for Drone (e.g., `drone.example.com`)
- `helm` and `kubectl` installed

## Step 1: Create a GitHub OAuth Application

1. Navigate to **GitHub → Settings → Developer Settings → OAuth Apps → New OAuth App**.
2. Fill in:
   - **Application name**: Drone CI
   - **Homepage URL**: `https://drone.example.com`
   - **Authorization callback URL**: `https://drone.example.com/login`
3. Copy the **Client ID** and **Client Secret**.

## Step 2: Create Drone Secrets

```bash
# Generate a shared secret between Drone server and runners
DRONE_RPC_SECRET=$(openssl rand -hex 32)
echo "Drone RPC Secret: ${DRONE_RPC_SECRET}"

# Store secrets in Kubernetes
kubectl create namespace drone

kubectl create secret generic drone-secrets \
  -n drone \
  --from-literal=DRONE_GITHUB_CLIENT_ID=<your-github-client-id> \
  --from-literal=DRONE_GITHUB_CLIENT_SECRET=<your-github-client-secret> \
  --from-literal=DRONE_RPC_SECRET="${DRONE_RPC_SECRET}"
```

## Step 3: Install Drone Server

```bash
# Add the Drone Helm repository
helm repo add drone https://charts.drone.io
helm repo update

# Create values for the Drone server
cat << 'EOF' > drone-server-values.yaml
image:
  tag: "2"

env:
  DRONE_GITHUB_SERVER: https://github.com
  DRONE_SERVER_HOST: drone.example.com
  DRONE_SERVER_PROTO: https
  DRONE_DATABASE_DRIVER: sqlite3
  DRONE_DATABASE_DATASOURCE: /data/database.sqlite

# Reference the secrets we created
envFrom:
  - secretRef:
      name: drone-secrets

persistence:
  enabled: true
  size: 10Gi
  storageClass: default

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: drone.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: drone-tls
      hosts:
        - drone.example.com
EOF

helm install drone-server drone/drone \
  --namespace drone \
  -f drone-server-values.yaml
```

## Step 4: Install Drone Kubernetes Runner

The Kubernetes runner executes pipeline steps as Kubernetes pods:

```bash
cat << 'EOF' > drone-runner-values.yaml
image:
  tag: latest

rbac:
  buildNamespaces:
    - drone-builds

env:
  DRONE_RPC_HOST: drone.example.com
  DRONE_RPC_PROTO: https
  DRONE_NAMESPACE_DEFAULT: drone-builds
  DRONE_RESOURCE_REQUEST_CPU: "100m"
  DRONE_RESOURCE_REQUEST_MEMORY: "128Mi"
  DRONE_RESOURCE_LIMIT_CPU: "1"
  DRONE_RESOURCE_LIMIT_MEMORY: "1Gi"

envFrom:
  - secretRef:
      name: drone-secrets

replicaCount: 2
EOF

# Create the build namespace
kubectl create namespace drone-builds

helm install drone-runner-kube drone/drone-runner-kube \
  --namespace drone \
  -f drone-runner-values.yaml
```

## Step 5: Create a Basic Drone Pipeline

```yaml
# .drone.yml — in the root of your application repository
kind: pipeline
type: kubernetes
name: default

steps:
  # Step 1: Run unit tests
  - name: test
    image: golang:1.21
    commands:
      - go test ./...

  # Step 2: Build and push Docker image
  - name: build-push
    image: plugins/docker
    settings:
      repo: ghcr.io/my-org/myapp
      registry: ghcr.io
      tags:
        - ${DRONE_COMMIT_SHA:0:8}
        - latest
      username:
        from_secret: docker_username
      password:
        from_secret: docker_password
    when:
      branch:
        - main

  # Step 3: Deploy to Rancher cluster
  - name: deploy
    image: bitnami/kubectl:latest
    environment:
      KUBECONFIG_DATA:
        from_secret: rancher_kubeconfig
    commands:
      - echo "$KUBECONFIG_DATA" | base64 -d > /tmp/kubeconfig
      - >
        kubectl set image deployment/myapp
        myapp=ghcr.io/my-org/myapp:${DRONE_COMMIT_SHA:0:8}
        -n production
        --kubeconfig /tmp/kubeconfig
      - kubectl rollout status deployment/myapp -n production --kubeconfig /tmp/kubeconfig
    when:
      branch:
        - main
```

## Step 6: Store Drone Secrets for Deployments

```bash
# Store the Rancher kubeconfig as a Drone secret
# First, base64-encode the kubeconfig
KUBECONFIG_B64=$(cat /path/to/rancher-cluster.kubeconfig | base64 -w 0)

# In Drone UI: Repository → Settings → Secrets → Add Secret
# Name: rancher_kubeconfig
# Value: <KUBECONFIG_B64>

# Or via Drone CLI:
drone secret add \
  --repository my-org/myapp \
  --name rancher_kubeconfig \
  --data "${KUBECONFIG_B64}"
```

## Step 7: Configure Multi-Stage Pipelines

```yaml
# .drone.yml — Multi-stage with environment promotion
kind: pipeline
type: kubernetes
name: staging

steps:
  - name: deploy-staging
    image: bitnami/kubectl:latest
    environment:
      KUBECONFIG_DATA:
        from_secret: kubeconfig_staging
    commands:
      - echo "$KUBECONFIG_DATA" | base64 -d > /tmp/kubeconfig
      - kubectl set image deployment/myapp myapp=ghcr.io/my-org/myapp:${DRONE_COMMIT_SHA:0:8} -n staging --kubeconfig /tmp/kubeconfig

---
kind: pipeline
type: kubernetes
name: production

# Require manual promotion from staging
trigger:
  event:
    - promote
  target:
    - production

steps:
  - name: deploy-production
    image: bitnami/kubectl:latest
    environment:
      KUBECONFIG_DATA:
        from_secret: kubeconfig_prod
    commands:
      - echo "$KUBECONFIG_DATA" | base64 -d > /tmp/kubeconfig
      - kubectl set image deployment/myapp myapp=ghcr.io/my-org/myapp:${DRONE_BUILD_PARENT_SHA:0:8} -n production --kubeconfig /tmp/kubeconfig
```

## Conclusion

Drone CI on Rancher provides a lightweight, container-native CI/CD experience that integrates well with the Kubernetes ecosystem. The Kubernetes runner ensures builds scale with your cluster resources, and Drone's simple YAML pipeline format reduces CI configuration complexity. Combined with Rancher's multi-cluster management, you can easily extend Drone pipelines to deploy to any cluster in your fleet.
