# How to Set Up Harbor with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Harbor, Container Registry

Description: A step-by-step guide to deploying Harbor as your private container registry and integrating it with Rancher for enterprise-grade image management.

## Introduction

Harbor is an open-source cloud-native container registry that provides security scanning, replication, access control, and audit logging. Integrating Harbor with Rancher gives your teams a fully featured, self-hosted registry with enterprise capabilities. This guide covers deploying Harbor on Kubernetes via Helm and integrating it with Rancher clusters.

## Prerequisites

- Rancher v2.6+ with an existing downstream cluster
- Helm 3.x installed
- A StorageClass for persistent volumes
- A domain name or IP for Harbor (with TLS certificate recommended)
- cert-manager installed (optional, for automatic TLS)

## Step 1: Add the Harbor Helm Repository

```bash
# Add the Harbor Helm chart repository
helm repo add harbor https://helm.goharbor.io
helm repo update

# Search for available Harbor versions
helm search repo harbor/harbor
```

## Step 2: Create Namespace and Configure Values

```bash
# Create dedicated namespace for Harbor
kubectl create namespace harbor
```

Create a custom values file:

```yaml
# harbor-values.yaml - Production-grade Harbor configuration
expose:
  type: ingress
  tls:
    enabled: true
    certSource: secret
    secret:
      # Reference your TLS secret
      secretName: harbor-tls
  ingress:
    hosts:
      core: harbor.example.com
    className: nginx
    annotations:
      nginx.ingress.kubernetes.io/proxy-body-size: "0"
      nginx.ingress.kubernetes.io/proxy-read-timeout: "600"

# External URL used to access Harbor
externalURL: https://harbor.example.com

# Persistent storage configuration
persistence:
  enabled: true
  resourcePolicy: "keep"
  persistentVolumeClaim:
    registry:
      storageClass: "standard"
      size: 100Gi
    jobservice:
      storageClass: "standard"
      size: 10Gi
    database:
      storageClass: "standard"
      size: 10Gi
    redis:
      storageClass: "standard"
      size: 5Gi
    trivy:
      storageClass: "standard"
      size: 5Gi

# Admin password - change this in production
harborAdminPassword: "Harbor12345"

# Enable Trivy for vulnerability scanning
trivy:
  enabled: true
  # Auto-update Trivy DB
  autoUpdate: true

# Database configuration
database:
  type: internal
  internal:
    password: "changeme"

# Redis configuration
redis:
  type: internal

# Notary for image signing
notary:
  enabled: true
```

## Step 3: Deploy Harbor with Helm

```bash
# Install Harbor using the custom values
helm install harbor harbor/harbor \
  --namespace harbor \
  --values harbor-values.yaml \
  --wait \
  --timeout 10m

# Check deployment status
kubectl get pods -n harbor
kubectl get ingress -n harbor
```

## Step 4: Create a TLS Certificate

```yaml
# harbor-certificate.yaml - TLS certificate via cert-manager
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: harbor-tls
  namespace: harbor
spec:
  secretName: harbor-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - harbor.example.com
```

```bash
kubectl apply -f harbor-certificate.yaml
```

## Step 5: Configure Harbor Projects and Users

After Harbor is running, log in at `https://harbor.example.com` with the admin credentials.

Create a project via the Harbor API:

```bash
# Create a project via Harbor API
curl -X POST "https://harbor.example.com/api/v2.0/projects" \
  -H "Content-Type: application/json" \
  -u admin:Harbor12345 \
  -d '{
    "project_name": "production",
    "public": false,
    "metadata": {
      "enable_content_trust": "true",
      "prevent_vul": "true",
      "severity": "high",
      "auto_scan": "true"
    }
  }'
```

## Step 6: Integrate Harbor with Rancher

Create registry credentials in Rancher pointing to Harbor:

```bash
# Create a secret for Harbor authentication in your workload namespace
kubectl create secret docker-registry harbor-credentials \
  --docker-server=harbor.example.com \
  --docker-username=robot$ci-user \
  --docker-password=<robot-account-token> \
  --namespace=production
```

Or configure Harbor as the cluster-level registry in Rancher:

```yaml
# Add to your RKE2 cluster configuration
registries:
  configs:
    harbor.example.com:
      authConfigSecretName: harbor-credentials
      caBundle: |
        -----BEGIN CERTIFICATE-----
        # Your CA certificate here
        -----END CERTIFICATE-----
```

## Step 7: Enable Vulnerability Scanning in CI/CD

Configure Harbor to auto-scan images on push:

```bash
# Enable auto-scan for a project via API
curl -X PUT "https://harbor.example.com/api/v2.0/projects/production" \
  -H "Content-Type: application/json" \
  -u admin:Harbor12345 \
  -d '{"metadata": {"auto_scan": "true"}}'

# Set policy to prevent deployment of vulnerable images
curl -X PUT "https://harbor.example.com/api/v2.0/projects/production" \
  -H "Content-Type: application/json" \
  -u admin:Harbor12345 \
  -d '{"metadata": {"prevent_vul": "true", "severity": "high"}}'
```

## Step 8: Configure Harbor Replication

Set up replication between Harbor and Docker Hub or other registries:

```bash
# Create a replication rule via API
curl -X POST "https://harbor.example.com/api/v2.0/replication/policies" \
  -H "Content-Type: application/json" \
  -u admin:Harbor12345 \
  -d '{
    "name": "sync-from-dockerhub",
    "src_registry": {"id": 1},
    "dest_namespace": "mirror",
    "filters": [{"type": "name", "value": "library/**"}],
    "trigger": {"type": "scheduled", "trigger_settings": {"cron": "0 0 * * *"}},
    "enabled": true
  }'
```

## Troubleshooting

```bash
# Check Harbor pod logs for issues
kubectl logs -n harbor -l component=core --tail=100

# Verify all Harbor components are running
kubectl get pods -n harbor -o wide

# Test Harbor connectivity
curl -I https://harbor.example.com/api/v2.0/ping
```

## Conclusion

Harbor provides a robust, enterprise-grade private registry solution that integrates seamlessly with Rancher. With built-in vulnerability scanning, content trust, replication, and RBAC, Harbor is an excellent choice for organizations managing containerized workloads at scale. Combined with Rancher's multi-cluster management capabilities, you get a complete container lifecycle management platform.
