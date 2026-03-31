# Best Practices for Registry Management in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Registry, Container Images, Security, Best Practice, CI/CD

Description: Manage Docker image registries in Portainer securely with proper credential storage, registry access policies, image scanning, and pull policies for production deployments.

---

Container registries are where your application images live. Managing them correctly in Portainer involves secure credential storage, access policies, and image hygiene practices that prevent security incidents and deployment failures.

## Registry Types Supported

Portainer supports multiple registry types:

- Docker Hub (public and private)
- Custom Docker registries (self-hosted)
- AWS ECR (Elastic Container Registry)
- Azure Container Registry (ACR)
- Google Artifact Registry
- Quay.io
- GitHub Container Registry (GHCR)

## Adding a Registry Securely

Add registries via **Registries > Add Registry**. For credentials:

```bash
# Do NOT use personal credentials for shared Portainer instances

# Create a dedicated service account/robot account in your registry

# Docker Hub: Create a Personal Access Token (not your password)
# ECR: Create an IAM role or service account with registry read permissions
# GHCR: Create a GitHub Personal Access Token with packages:read scope
# Private registry: Create a robot account with read-only access
```

For AWS ECR, credentials rotate every 12 hours. Use the Portainer ECR registry type which handles token refresh automatically.

## Registry Access Policies (BE)

Portainer Business Edition allows restricting which registries teams can pull from:

1. Go to **Registries > [Registry Name] > Access**
2. Grant specific teams access
3. Teams without access cannot pull from that registry

This prevents developers from deploying arbitrary images from Docker Hub in production.

## Self-Hosted Registry Deployment

Deploy a private registry alongside Portainer for complete control:

```yaml
# private-registry-stack.yml
version: "3.8"

services:
  registry:
    image: registry:2.8
    environment:
      - REGISTRY_AUTH=htpasswd
      - REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd
      - REGISTRY_AUTH_HTPASSWD_REALM=Registry
      - REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt
      - REGISTRY_HTTP_TLS_KEY=/certs/registry.key
      # Enable S3 storage for scalability
      # - REGISTRY_STORAGE=s3
      # - REGISTRY_STORAGE_S3_BUCKET=my-registry
    volumes:
      - /opt/registry/data:/var/lib/registry
      - /opt/registry/auth:/auth:ro
      - /opt/registry/certs:/certs:ro
    ports:
      - "5000:5000"
    restart: unless-stopped

volumes:
  registry-data:
```

## Image Scanning Integration (BE)

Portainer BE integrates with Trivy for vulnerability scanning:

1. Enable image scanning in **Settings > Security**
2. Scan images before deployment by checking results in the image details view
3. Set policy: block deployment of images with CRITICAL vulnerabilities

## Pull Policies

Set appropriate pull policies in your stacks:

```yaml
# Always pull in production to catch updated security patches
services:
  app:
    image: myregistry/app:1.2.3
    pull_policy: always    # Pull on every container start

# For development, use IfNotPresent to save bandwidth
services:
  app:
    image: mydev-registry/app:latest
    pull_policy: if_not_present
```

## Image Pruning

Schedule regular image cleanup to manage disk space:

```bash
#!/bin/bash
# image-prune.sh - run as a Portainer scheduled job

# Remove dangling images (untagged layers)
docker image prune -f

# Remove images not used by any container (be careful in production)
# docker image prune -af --filter "until=168h"  # Older than 7 days

echo "Image cleanup complete. Disk usage:"
docker system df
```

## Registry Mirror for Bandwidth Optimization

Set up a registry pull-through cache to reduce Docker Hub bandwidth:

```yaml
# registry-mirror-stack.yml
services:
  registry-mirror:
    image: registry:2.8
    environment:
      # Configure as a pull-through cache for Docker Hub
      - REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io
    volumes:
      - registry-mirror-data:/var/lib/registry
    ports:
      - "5001:5000"
```

Configure Docker daemon on your hosts to use the mirror:

```json
// /etc/docker/daemon.json
{
  "registry-mirrors": ["https://your-mirror.example.com:5001"]
}
```

## Summary

Registry management in Portainer requires dedicated service accounts for credentials, access policies to restrict deployment sources, image scanning to catch vulnerabilities, and regular cleanup to manage disk usage. These practices prevent supply chain attacks and keep your image infrastructure lean.
