# How to Configure Application Annotations in Portainer for Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Annotations, Ingress, Configuration

Description: Learn how to add and manage Kubernetes annotations on applications deployed through Portainer.

## What Are Kubernetes Annotations?

Annotations are key-value pairs attached to Kubernetes objects that store non-identifying metadata. Unlike labels (used for selection), annotations hold configuration hints, tool-specific settings, or documentation. Common uses:

- Ingress controller configuration (Nginx, Traefik)
- Prometheus scraping instructions
- Deployment strategy hints
- GitOps metadata

## Adding Annotations in Portainer

When creating or editing an application in Portainer:

1. Scroll to the **Advanced configuration** section.
2. Find **Annotations** or **Labels and annotations**.
3. Click **Add annotation** and enter key-value pairs.
4. Click **Deploy** or **Update**.

## Common Annotation Examples

### Nginx Ingress Annotations

```yaml
# Deployment metadata annotations

metadata:
  name: my-app
  annotations:
    # Nginx Ingress controller configuration
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
```

### Prometheus Scraping Annotations

```yaml
# Pod template annotations for Prometheus auto-discovery
metadata:
  annotations:
    prometheus.io/scrape: "true"       # Tell Prometheus to scrape this pod
    prometheus.io/port: "8080"         # Port where metrics are exposed
    prometheus.io/path: "/metrics"     # Metrics endpoint path
```

### Deployment Change Tracking

```yaml
metadata:
  annotations:
    # Track who deployed and from which CI job
    deployment.kubernetes.io/deployed-by: "github-actions"
    deployment.kubernetes.io/git-commit: "abc123"
    deployment.kubernetes.io/deployed-at: "2026-03-20T10:30:00Z"
```

## Force Pod Restart via Annotation

You can trigger a rolling restart without changing the application logic by updating an annotation:

```bash
# Trigger a rolling restart by updating an annotation
kubectl annotate deployment my-app \
  restart-time="$(date +%s)" \
  --overwrite \
  --namespace=production

# Or use the dedicated restart command
kubectl rollout restart deployment/my-app --namespace=production
```

## Adding Annotations via CLI

```bash
# Add an annotation to a deployment
kubectl annotate deployment my-app \
  description="Primary web application" \
  --namespace=production

# Update an existing annotation
kubectl annotate deployment my-app \
  description="Updated web application" \
  --overwrite \
  --namespace=production

# Remove an annotation
kubectl annotate deployment my-app \
  description- \
  --namespace=production

# View all annotations
kubectl get deployment my-app -o jsonpath='{.metadata.annotations}' \
  --namespace=production | jq
```

## Ingress with Annotations Example

```yaml
# Ingress with Nginx-specific annotations
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts: [myapp.example.com]
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

## Conclusion

Annotations are a powerful metadata mechanism in Kubernetes. Portainer's annotation editor lets you add the necessary hints for ingress controllers, monitoring systems, and GitOps tooling without switching to the command line.
