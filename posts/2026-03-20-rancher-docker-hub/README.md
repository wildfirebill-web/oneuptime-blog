# How to Configure Docker Hub Integration in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Docker Hub, Container Registry

Description: Configure Docker Hub integration in Rancher to avoid rate limits, authenticate with private repositories, and manage image pulls across your clusters.

## Introduction

Docker Hub is the world's largest container image registry, hosting millions of public and private images. However, Docker Hub enforces rate limits on unauthenticated pulls, which can cause issues in production Kubernetes environments. Integrating Docker Hub credentials with Rancher ensures authenticated pulls, access to private repositories, and higher rate limits.

## Prerequisites

- A Rancher instance managing at least one cluster
- A Docker Hub account (free or paid)
- Docker Hub access token (recommended over password)

## Step 1: Create a Docker Hub Access Token

Using an access token is more secure than using your password:

1. Log in to [Docker Hub](https://hub.docker.com).
2. Go to **Account Settings** > **Security** > **New Access Token**.
3. Give it a name like `rancher-cluster` and select **Read-only** permissions.
4. Copy the token (you won't see it again).

## Step 2: Add Docker Hub Credentials in Rancher UI

1. In Rancher, navigate to your cluster or project.
2. Go to **Secrets** > **Registry Credentials**.
3. Click **Add Registry**.
4. Configure the form:
   - **Name**: `docker-hub-credentials`
   - **Registry**: Select **DockerHub** (or enter `docker.io`)
   - **Username**: Your Docker Hub username
   - **Password**: Your access token

## Step 3: Create Docker Hub Secret via kubectl

```bash
# Create Docker Hub credentials as a Kubernetes secret

kubectl create secret docker-registry docker-hub-credentials \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myusername \
  --docker-password=dckr_pat_xxxxxxxxxxxx \
  --docker-email=myemail@example.com \
  --namespace=default
```

Verify the secret:

```bash
# View the secret (base64 encoded)
kubectl get secret docker-hub-credentials -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .
```

## Step 4: Use Docker Hub Credentials in Workloads

```yaml
# deployment.yaml - Using Docker Hub private image
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-private-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-private-app
  template:
    metadata:
      labels:
        app: my-private-app
    spec:
      # Reference the Docker Hub credentials
      imagePullSecrets:
        - name: docker-hub-credentials
      containers:
        - name: app
          # Private Docker Hub image
          image: myusername/private-app:latest
          ports:
            - containerPort: 3000
```

## Step 5: Apply Credentials Cluster-Wide

To avoid adding `imagePullSecrets` to every deployment, patch the default service account:

```bash
# Apply to the default service account in your namespace
kubectl patch serviceaccount default \
  --namespace default \
  --type merge \
  --patch '{"imagePullSecrets": [{"name": "docker-hub-credentials"}]}'

# Verify the patch
kubectl get serviceaccount default -o yaml
```

## Step 6: Configure Docker Hub Mirror to Reduce Rate Limits

For high-volume clusters, configure Docker Hub as a mirror through a pull-through cache:

```yaml
# registries.yaml - Configure Docker Hub mirror for RKE2 nodes
mirrors:
  docker.io:
    endpoints:
      - "https://mirror.gcr.io"       # Google's Docker Hub mirror
      - "https://registry-1.docker.io" # Fallback to Docker Hub directly
configs:
  "registry-1.docker.io":
    auth:
      username: myusername
      password: dckr_pat_xxxxxxxxxxxx
```

Deploy this configuration to each RKE2 node:

```bash
# Copy the registries configuration to each node
sudo cp registries.yaml /etc/rancher/rke2/registries.yaml
sudo systemctl restart rke2-server
```

## Step 7: Monitor Docker Hub Pull Rate Usage

```bash
# Check current rate limit status (replace with your token)
TOKEN=$(curl -s -H "Content-Type: application/json" \
  -X POST \
  -d '{"username": "myusername", "password": "dckr_pat_xxxx"}' \
  https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull \
  | jq -r .token)

# Check rate limit headers
curl -s --head \
  -H "Authorization: Bearer ${TOKEN}" \
  https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest \
  | grep -i ratelimit
```

## Step 8: Use Docker Hub with Rancher Fleet

For GitOps deployments with Fleet, reference Docker Hub credentials:

```yaml
# fleet.yaml - Fleet bundle using Docker Hub
defaultNamespace: production
helm:
  releaseName: my-app
  values:
    image:
      repository: myusername/my-app
      tag: v1.2.0
    imagePullSecrets:
      - name: docker-hub-credentials
```

## Troubleshooting

### Too Many Requests (429) Error

```bash
# Check if you're hitting rate limits
kubectl describe pod <failing-pod> | grep -A 5 "Failed to pull image"

# Solution: ensure imagePullSecrets is configured correctly
kubectl get pod <failing-pod> -o jsonpath='{.spec.imagePullSecrets}'
```

### Authentication Failures

```bash
# Verify credentials work locally
docker login -u myusername
docker pull myusername/private-image:latest

# Test in-cluster authentication
kubectl run test --image=myusername/private-image:latest \
  --overrides='{"spec":{"imagePullSecrets":[{"name":"docker-hub-credentials"}]}}' \
  --restart=Never
```

## Conclusion

Integrating Docker Hub with Rancher ensures reliable image pulls, avoids rate-limiting issues, and enables access to private repositories. For production environments, use access tokens instead of passwords, configure pull-through caches for frequently used public images, and monitor your Docker Hub rate limit usage to prevent disruptions.
