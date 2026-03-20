# How to Install Epinio from Rancher UI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Epinio, Rancher, Kubernetes, PaaS, Apps

Description: Install and configure Epinio directly from the Rancher UI using the Apps & Marketplace for a seamless developer platform setup.

## Introduction

Rancher's App Marketplace makes installing Epinio as simple as clicking through a wizard. This approach is ideal for platform teams that manage Kubernetes through Rancher and want to offer a PaaS experience to their development teams without command-line tools.

## Prerequisites

- Rancher v2.7+
- A downstream Kubernetes cluster managed by Rancher
- A wildcard DNS domain pointed to your cluster's ingress IP
- cert-manager installed on the target cluster

## Step 1: Access Apps & Marketplace

1. Log into Rancher
2. Select the target cluster from the cluster selector
3. Navigate to **Apps** > **Charts** in the left sidebar
4. Search for **Epinio** in the chart listing

## Step 2: Configure Epinio Installation

In the Epinio chart configuration screen:

### Basic Settings

```yaml
# These are configured through the Rancher UI form:
global:
  domain: "epinio.example.com"      # Your domain
  tlsIssuer: "letsencrypt-production"  # Or selfsigned-issuer

# Storage backend (MinIO is included by default)
minio:
  enabled: true
  persistence:
    size: 20Gi
```

### Advanced Settings

```yaml
# Custom container registry settings
containerregistry:
  enabled: true
  persistence:
    size: 20Gi
    storageClass: "longhorn"

# Dex (OIDC) configuration for SSO
dex:
  enabled: true
  config:
    connectors:
      - type: ldap
        id: ldap
        name: "Corporate LDAP"
        config:
          host: ldap.example.com:636

# Resource limits
epinio-ui:
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

## Step 3: Install via Rancher UI

1. Click **Install** on the Epinio chart page
2. Select the **epinio** namespace (or create it)
3. Fill in the configuration form:
   - **Domain**: `epinio.example.com`
   - **TLS Issuer**: `letsencrypt-production`
4. Click **Next** to review the YAML
5. Click **Install** to begin installation

## Step 4: Monitor Installation

In the Rancher UI:

1. Navigate to **Apps** > **Installed Apps**
2. Find the **epinio** app
3. Watch the installation progress
4. Wait for all pods to show **Running** status

```bash
# Also monitor from CLI
kubectl get pods -n epinio --watch
```

## Step 5: Configure DNS

After installation, get the ingress IP:

```bash
# Get the external IP of the ingress controller
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Configure DNS:
- `epinio.example.com` → ingress IP
- `*.epinio.example.com` → ingress IP (wildcard for app domains)

## Step 6: Access Epinio UI

1. Navigate to `https://epinio.example.com` in your browser
2. Log in with admin credentials
3. The Epinio dashboard shows namespaces, applications, and services

## Step 7: Install Epinio CLI for Developers

```bash
# Give developers the CLI for push workflows
curl -fsSL \
  https://github.com/epinio/epinio/releases/latest/download/epinio-linux-x86_64 \
  -o /usr/local/bin/epinio
chmod +x /usr/local/bin/epinio

# Login with Rancher-managed credentials
epinio login https://epinio.example.com \
  --user admin \
  --password "your-password"
```

## Upgrading Epinio via Rancher UI

1. Navigate to **Apps** > **Installed Apps**
2. Click on the **epinio** app
3. Click **Upgrade**
4. Review and update any changed values
5. Click **Upgrade** to apply

## Conclusion

Installing Epinio from Rancher UI provides an integrated experience for platform teams already using Rancher for cluster management. The visual configuration form makes it easy to get started, while the advanced YAML editor gives you full control over the deployment. Once installed, Epinio appears as a managed application in Rancher, making upgrades and configuration changes straightforward.
