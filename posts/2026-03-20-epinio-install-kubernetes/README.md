# How to Install Epinio on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Epinio, Kubernetes, PaaS, Developer Experience, Helm

Description: A complete guide to installing Epinio, the application deployment platform, on any Kubernetes cluster using Helm.

## Introduction

Epinio is a developer-friendly PaaS (Platform as a Service) built on Kubernetes. It provides a simple push-to-deploy workflow similar to Heroku or Cloud Foundry, while running on your own Kubernetes cluster. Developers can deploy applications without writing Kubernetes manifests, while operators maintain full control over the underlying infrastructure.

## Prerequisites

- Kubernetes cluster (v1.24+)
- `kubectl` configured for the cluster
- `helm` v3.x installed
- A wildcard domain or ingress controller
- cert-manager (optional but recommended)

## Step 1: Install Required Dependencies

### Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

### Install an Ingress Controller

```bash
# Install NGINX Ingress
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Get the external IP
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

## Step 2: Add the Epinio Helm Repository

```bash
helm repo add epinio https://epinio.github.io/helm-charts
helm repo update

# See available versions
helm search repo epinio
```

## Step 3: Install Epinio

### Basic Installation

```bash
# Install Epinio with your domain
helm install epinio epinio/epinio \
  --namespace epinio \
  --create-namespace \
  --set global.domain=epinio.example.com \
  --set global.tlsIssuer=selfsigned-issuer
```

### Production Installation with Let's Encrypt

```yaml
# epinio-values.yaml
global:
  # Your wildcard domain (e.g., *.epinio.example.com)
  domain: epinio.example.com
  # TLS configuration
  tlsIssuer: letsencrypt-production

# Enable S3-compatible storage
s3:
  endpoint: s3.amazonaws.com
  bucket: my-epinio-bucket
  region: us-west-2
  accessKeyId: "your-access-key"
  secretAccessKey: "your-secret-key"

# Configure internal container registry
containerregistry:
  enabled: true
```

```bash
helm install epinio epinio/epinio \
  --namespace epinio \
  --create-namespace \
  --values epinio-values.yaml
```

## Step 4: Install the Epinio CLI

```bash
# Install on Linux (AMD64)
curl -fsSL \
  https://github.com/epinio/epinio/releases/latest/download/epinio-linux-x86_64 \
  -o /usr/local/bin/epinio
chmod +x /usr/local/bin/epinio

# Verify installation
epinio version
```

## Step 5: Login to Epinio

```bash
# Get the Epinio admin password
EPINIO_PASSWORD=$(kubectl get secret epinio-creds \
  -n epinio \
  -o jsonpath='{.data.password}' | base64 -d)

# Login
epinio login https://epinio.epinio.example.com \
  --user admin \
  --password "${EPINIO_PASSWORD}"

# Verify login
epinio info
```

## Step 6: Verify Installation

```bash
# Check all Epinio pods are running
kubectl get pods -n epinio

# Check Epinio services
kubectl get svc -n epinio

# Verify ingress is configured
kubectl get ingress -n epinio

# Test the API
epinio app list
```

## Step 7: Deploy a Test Application

```bash
# Create a test application directory
mkdir hello-world && cd hello-world
cat > app.rb << 'EOF'
require 'sinatra'
get '/' do
  "Hello from Epinio!\n"
end
EOF

# Push the application
epinio push --name hello-world

# Check deployment
epinio app show hello-world
```

## Conclusion

Epinio transforms Kubernetes into a developer-friendly PaaS without sacrificing operator control. Once installed, developers can push applications using a simple `epinio push` command without needing to understand Kubernetes internals. The platform handles buildpacks, containers, routing, and SSL certificates automatically.
