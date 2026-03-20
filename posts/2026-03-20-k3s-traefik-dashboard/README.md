# How to Configure Traefik Dashboard in K3s

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Traefik, Monitoring, Dashboards, DevOps

Description: Learn how to enable, secure, and access the Traefik dashboard in K3s for monitoring ingress routing and service health.

## Introduction

The Traefik dashboard provides a real-time web UI showing all configured routers, services, middlewares, and their health status. While Traefik ships with K3s, its dashboard is not enabled by default for security reasons. This guide covers enabling the dashboard, securing it with authentication, and exposing it via an IngressRoute.

## Prerequisites

- K3s running with Traefik (default installation)
- `kubectl` configured
- Basic understanding of Traefik

## Step 1: Enable the Traefik Dashboard via HelmChartConfig

Use K3s's `HelmChartConfig` CRD to customize Traefik without overriding the entire chart:

```yaml
# /var/lib/rancher/k3s/server/manifests/traefik-dashboard-config.yaml

apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    dashboard:
      enabled: true
    # Enable the Traefik API (required for dashboard)
    api:
      dashboard: true
      insecure: false  # Never use insecure mode in production
    logs:
      general:
        level: ERROR
      access:
        enabled: true
    metrics:
      prometheus:
        enabled: true
```

Apply the config:

```bash
# K3s auto-detects changes in the manifests directory
# Or manually apply
kubectl apply -f /var/lib/rancher/k3s/server/manifests/traefik-dashboard-config.yaml

# Wait for Traefik to restart and apply the new config
kubectl rollout status deployment/traefik -n kube-system
```

## Step 2: Create Dashboard Authentication

Never expose the dashboard without authentication:

```bash
# Install apache2-utils for htpasswd
apt-get install -y apache2-utils

# Create a secure password (replace 'admin' and 'securepassword')
htpasswd -nb admin 'S3cur3P@ssw0rd!' > /tmp/dashboard-auth.txt

# View the generated hash
cat /tmp/dashboard-auth.txt
# Output: admin:$2y$10$...

# Create Kubernetes secret for basic auth
kubectl create secret generic traefik-dashboard-auth \
  --from-file=users=/tmp/dashboard-auth.txt \
  -n kube-system

# Clean up the temp file
rm /tmp/dashboard-auth.txt
```

## Step 3: Create Middleware for Authentication

```yaml
# /var/lib/rancher/k3s/server/manifests/traefik-dashboard-middleware.yaml
---
# Basic authentication middleware for the dashboard
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: dashboard-auth
  namespace: kube-system
spec:
  basicAuth:
    secret: traefik-dashboard-auth
    removeHeader: true   # Remove auth header before forwarding to backend
---
# IP whitelist middleware (restrict to internal network)
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: dashboard-ip-whitelist
  namespace: kube-system
spec:
  ipWhiteList:
    sourceRange:
      - "10.0.0.0/8"
      - "172.16.0.0/12"
      - "192.168.0.0/16"
      - "127.0.0.1/32"
```

## Step 4: Expose Dashboard via IngressRoute

```yaml
# /var/lib/rancher/k3s/server/manifests/traefik-dashboard-route.yaml
---
# HTTP route that redirects to HTTPS
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard-http
  namespace: kube-system
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`traefik.example.com`)
      kind: Rule
      middlewares:
        - name: https-redirect
          namespace: kube-system
      services:
        - name: api@internal
          kind: TraefikService
---
# HTTPS route with authentication
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard-https
  namespace: kube-system
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.example.com`) && (PathPrefix(`/dashboard`) || PathPrefix(`/api`))
      kind: Rule
      middlewares:
        - name: dashboard-auth
          namespace: kube-system
        - name: dashboard-ip-whitelist
          namespace: kube-system
      services:
        # api@internal is Traefik's internal API service
        - name: api@internal
          kind: TraefikService
  tls:
    secretName: traefik-dashboard-tls
```

## Step 5: Option B - Access Dashboard via Port-Forward

For quick access without exposing the dashboard publicly:

```bash
# Port-forward to the Traefik service
kubectl port-forward -n kube-system \
  service/traefik 9000:9000 &

# Access the dashboard at:
# http://localhost:9000/dashboard/

# Or forward to the pod directly
TRAEFIK_POD=$(kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik \
  -o jsonpath='{.items[0].metadata.name}')

kubectl port-forward -n kube-system pod/$TRAEFIK_POD 9000:9000

# Visit http://localhost:9000/dashboard/
```

## Step 6: Add HTTPS Redirect Middleware

```yaml
# /var/lib/rancher/k3s/server/manifests/https-redirect-middleware.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: https-redirect
  namespace: kube-system
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

## Step 7: Verify Dashboard is Working

```bash
# Check Traefik is running with new config
kubectl get pods -n kube-system | grep traefik

# Check Traefik logs for any errors
kubectl logs -n kube-system deployment/traefik --tail=50

# Verify the IngressRoute is registered
kubectl get ingressroutes -n kube-system

# Test dashboard access
curl -u admin:S3cur3P@ssw0rd! \
  -k https://traefik.example.com/dashboard/

# Check Traefik API
curl -u admin:S3cur3P@ssw0rd! \
  -k https://traefik.example.com/api/overview
```

## Step 8: Dashboard Features

Once accessed, the Traefik dashboard shows:

```text
Dashboard Sections:
├── Overview
│   ├── Total routers, services, middlewares
│   └── Health status summary
├── HTTP
│   ├── Routers - All HTTP routing rules
│   ├── Services - Backend services
│   └── Middlewares - Applied middleware
├── TCP
│   ├── Routers - TCP routing rules
│   └── Services - TCP backend services
└── UDP
    └── Routers and services for UDP
```

## Conclusion

The Traefik dashboard provides invaluable visibility into your K3s cluster's ingress routing. Enable it with HelmChartConfig for proper integration with K3s's Helm management, and always secure it with at least basic authentication and IP whitelisting. For production environments, restrict dashboard access to your internal network or VPN, and consider adding additional security layers like Traefik's forward authentication middleware for SSO integration.
