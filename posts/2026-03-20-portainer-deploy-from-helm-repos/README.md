# How to Deploy Applications from Helm Repositories in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Helm, Deployment, DevOps

Description: Learn how to browse Helm chart repositories in Portainer and deploy applications to Kubernetes clusters with customized values using the built-in Helm integration.

## Introduction

Portainer's Helm integration lets you deploy applications from public and private Helm repositories directly through the UI. This eliminates the need for local Helm CLI installations while providing a visual form to customize chart values before deploying. This guide covers the full deployment workflow from browsing to running applications.

## Prerequisites

- Portainer CE or BE with a Kubernetes environment
- At least one Helm repository configured (see the custom repo guide)
- A target namespace created in your cluster
- Admin or operator access to the namespace

## Step 1: Access the Helm Charts Catalog

1. Log into Portainer.
2. Select your **Kubernetes** environment.
3. Click **Helm** in the left sidebar.

You will see a searchable catalog of all charts from your configured repositories.

## Step 2: Browse and Search Charts

- Use the **Search** bar to find a specific chart (e.g., `nginx`, `postgresql`, `grafana`)
- Filter by **Repository** using the dropdown
- Each chart card shows:
  - Chart name and version
  - Description
  - Repository source
  - App version (the underlying application version)

## Step 3: Install a Helm Chart

1. Click the chart you want to install (e.g., `nginx` from the Bitnami repo).
2. Click **Install** on the chart detail page.
3. Fill in the deployment form:

   - **Release name**: A unique name for this deployment (e.g., `my-nginx`)
   - **Namespace**: Target namespace (e.g., `production`)
   - **Chart version**: Select the chart version to deploy

4. In the **Chart values** section, you can customize the deployment:
   - Edit directly in the YAML editor
   - Override specific values

```yaml
# Example: Customized NGINX values

replicaCount: 3

service:
  type: LoadBalancer
  port: 80

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

ingress:
  enabled: true
  hostname: nginx.example.com
  annotations:
    kubernetes.io/ingress.class: nginx
```

5. Click **Install**.

## Step 4: Deploy Common Charts with Example Values

### PostgreSQL

```yaml
# postgresql-values.yaml - Production-ready PostgreSQL
auth:
  postgresPassword: "changeme-secure-password"
  database: "myapp_db"
  username: "myapp_user"
  password: "changeme-app-password"

primary:
  persistence:
    enabled: true
    size: 20Gi
    storageClass: "local-path"

  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 250m
      memory: 256Mi
```

### Grafana

```yaml
# grafana-values.yaml
adminPassword: "changeme-grafana-password"

persistence:
  enabled: true
  size: 5Gi

ingress:
  enabled: true
  hosts:
    - grafana.example.com

service:
  type: ClusterIP
```

### cert-manager

```yaml
# cert-manager-values.yaml
installCRDs: true  # Install Custom Resource Definitions automatically

replicaCount: 2

resources:
  limits:
    cpu: 200m
    memory: 256Mi
```

## Step 5: Monitor the Installation

After clicking **Install**, Portainer shows the installation progress. You can:

1. Watch the pod status update in real time.
2. Check the **Helm releases** section for status: `deployed`, `pending`, `failed`.

```bash
# Check via KubeShell
kubectl get pods -n production
kubectl get svc -n production

# Check Helm release status
helm list -n production
```

## Step 6: Deploy via the Portainer API

For programmatic deployments:

```bash
TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' | jq -r '.jwt')

# Deploy nginx chart from Bitnami
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/helm" \
  -d '{
    "name": "my-nginx",
    "namespace": "production",
    "chart": "nginx",
    "version": "15.0.0",
    "repo": "https://charts.bitnami.com/bitnami",
    "values": "replicaCount: 2\nservice:\n  type: ClusterIP"
  }'
```

## Step 7: Upgrade a Helm Release

To update chart values or upgrade to a new version:

1. Go to **Helm** → **Releases**.
2. Click the release name.
3. Click **Upgrade**.
4. Modify the values or select a new chart version.
5. Click **Upgrade** to apply.

```bash
# Via CLI in KubeShell
helm upgrade my-nginx bitnami/nginx \
  --namespace production \
  --set replicaCount=5 \
  --reuse-values
```

## Conclusion

Deploying applications from Helm repositories in Portainer provides a visual, form-driven alternative to the Helm CLI while preserving full customization through YAML values editing. Start with sensible defaults, customize for your environment, and use the API for automated multi-environment deployments. Always pin chart versions in production to ensure reproducible deployments.
