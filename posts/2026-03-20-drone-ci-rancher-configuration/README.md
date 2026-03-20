# How to Configure Drone CI with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Drone CI, Rancher, Kubernetes, CI/CD, DevOps, Pipelines, Containers

Description: Learn how to deploy Drone CI on a Rancher-managed Kubernetes cluster and configure pipelines to build, test, and deploy containerized applications automatically.

---

Drone CI is a lightweight, container-native CI system that can run entirely on Kubernetes. This guide shows how to install Drone on Rancher and configure it for automated container deployments.

---

## Step 1: Create a GitHub OAuth Application

In GitHub, navigate to **Settings > Developer settings > OAuth Apps > New OAuth App**:

- **Homepage URL**: `https://drone.example.com`
- **Authorization callback URL**: `https://drone.example.com/login`

Note the **Client ID** and **Client Secret**.

---

## Step 2: Deploy Drone Server via Helm

```bash
helm repo add drone https://charts.drone.io
helm repo update

# Create a namespace for Drone
kubectl create namespace drone

# Create a secret with Drone credentials
kubectl create secret generic drone-secrets \
  --namespace drone \
  --from-literal=DRONE_GITHUB_CLIENT_ID=<your-client-id> \
  --from-literal=DRONE_GITHUB_CLIENT_SECRET=<your-client-secret> \
  --from-literal=DRONE_RPC_SECRET=$(openssl rand -hex 16)
```

Install the Drone server Helm chart with values that point to the secrets:

```yaml
# drone-values.yaml
env:
  DRONE_SERVER_HOST: drone.example.com
  DRONE_SERVER_PROTO: https
  DRONE_GITHUB_SERVER: https://github.com

# Reference credentials from the Kubernetes secret
envFrom:
  - secretRef:
      name: drone-secrets

persistentVolume:
  enabled: true
  size: 10Gi
```

```bash
helm install drone drone/drone \
  --namespace drone \
  --values drone-values.yaml
```

---

## Step 3: Deploy Drone Kubernetes Runner

The Kubernetes runner executes each pipeline step as a pod:

```bash
helm install drone-runner-kube drone/drone-runner-kube \
  --namespace drone \
  --set env.DRONE_RPC_HOST=drone-drone.drone.svc.cluster.local \
  --set env.DRONE_RPC_PROTO=http \
  --set env.DRONE_RPC_SECRET=<same-rpc-secret>
```

---

## Step 4: Write a Drone Pipeline

Drone pipelines are defined in `.drone.yml` at the root of your repository. Each step runs in its own container:

```yaml
# .drone.yml
kind: pipeline
type: kubernetes
name: default

# Runs on every push to main
trigger:
  branch:
    - main
  event:
    - push

steps:
  - name: test
    image: golang:1.22
    commands:
      # Run unit tests
      - go test ./...

  - name: build-image
    image: plugins/docker
    settings:
      # Push to Docker Hub using Drone secrets
      username:
        from_secret: docker_username
      password:
        from_secret: docker_password
      repo: my-org/my-app
      tags:
        - latest
        - ${DRONE_COMMIT_SHA:0:8}

  - name: deploy
    image: bitnami/kubectl:latest
    environment:
      KUBE_TOKEN:
        from_secret: kube_token
      KUBE_SERVER:
        from_secret: kube_server
    commands:
      # Configure kubectl and roll out new image
      - kubectl config set-cluster rancher --server=$KUBE_SERVER --insecure-skip-tls-verify=true
      - kubectl config set-credentials drone --token=$KUBE_TOKEN
      - kubectl config set-context rancher --cluster=rancher --user=drone
      - kubectl config use-context rancher
      - kubectl set image deployment/my-app app=my-org/my-app:${DRONE_COMMIT_SHA:0:8} -n my-app
      - kubectl rollout status deployment/my-app -n my-app
```

---

## Step 5: Add Secrets in Drone UI

In the Drone UI, go to your repository settings and add organization or repository secrets:

- `docker_username` / `docker_password`
- `kube_token` — the Rancher service account token
- `kube_server` — Rancher cluster API URL

---

## Best Practices

- Use **per-repo secrets** for application credentials and **org-level secrets** for shared infrastructure tokens.
- Set resource limits on the Kubernetes runner so pipeline pods don't starve production workloads.
- Enable Drone's **branch protection** filters to prevent untrusted forks from accessing secrets.
