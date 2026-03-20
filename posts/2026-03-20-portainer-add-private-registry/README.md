# How to Add a Custom Private Registry to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Registry, Private Registry, DevOps

Description: Learn how to add and configure a custom private container registry in Portainer for pulling images from self-hosted or corporate registries.

## Introduction

Organizations often run their own container registries for security, compliance, or latency reasons. Portainer supports adding custom private registries — whether self-hosted with Docker Registry, Harbor, Nexus Repository, or any other OCI-compliant registry. This guide covers configuring custom private registries in Portainer.

## Prerequisites

- Portainer CE or BE installed
- A running private registry with accessible URL
- Registry credentials (username/password or token)
- Admin access to Portainer

## Common Private Registry Options

| Registry | Notes |
|---------|-------|
| Docker Registry v2 | Simple, official Docker image |
| Harbor | Full-featured with RBAC and scanning |
| Nexus Repository | Maven + Docker + npm in one tool |
| JFrog Artifactory | Enterprise registry solution |
| Gitea + Package Registry | Built into Gitea |
| Self-hosted Docker Registry | Minimal footprint |

## Step 1: Deploy a Simple Private Registry (if needed)

If you don't already have a registry, deploy one quickly:

```yaml
# registry.yml
version: "3"
services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
    volumes:
      - registry-data:/var/lib/registry
      - /opt/registry/auth:/auth

volumes:
  registry-data:
```

```bash
# Create registry credentials
mkdir -p /opt/registry/auth
docker run --rm --entrypoint htpasswd httpd:2 \
  -Bbn registryuser strongpassword > /opt/registry/auth/htpasswd

# Start registry
docker compose -f registry.yml up -d
```

## Step 2: Add the Custom Registry in Portainer

1. In Portainer, click **Registries** in the sidebar
2. Click **+ Add registry**
3. Select **Custom registry**

## Step 3: Fill in Registry Details

```
Name:       My Private Registry
URL:        registry.company.com         (or IP:port like 10.0.0.5:5000)

Authentication:
  [x] Use authentication
  Username:   registryuser
  Password:   strongpassword
```

For registries using a different port:

```
URL: registry.company.com:5000
```

For insecure (HTTP) registries, you must also configure the Docker daemon:

```json
// /etc/docker/daemon.json on all Docker hosts
{
  "insecure-registries": ["registry.company.com:5000"]
}
```

4. Click **Add registry**

## Step 4: Configure TLS for HTTPS Registries

For production, always use HTTPS. If using a self-signed certificate:

```bash
# Add the CA cert to the Docker daemon on each host
sudo mkdir -p /etc/docker/certs.d/registry.company.com
sudo cp ca.crt /etc/docker/certs.d/registry.company.com/ca.crt

# Restart Docker to apply
sudo systemctl restart docker
```

Or configure the daemon to trust the CA system-wide:

```bash
sudo cp ca.crt /usr/local/share/ca-certificates/company-registry.crt
sudo update-ca-certificates
sudo systemctl restart docker
```

## Step 5: Push and Pull Images

```bash
# Tag an image for your private registry
docker tag myapp:latest registry.company.com/team/myapp:latest

# Push to private registry
docker push registry.company.com/team/myapp:latest

# Pull from private registry
docker pull registry.company.com/team/myapp:latest
```

## Step 6: Use Private Images in Portainer Stacks

In your Compose file, reference the private registry image:

```yaml
version: "3.8"

services:
  app:
    image: registry.company.com/team/myapp:latest
    # Portainer uses the stored registry credentials automatically

  db-proxy:
    image: registry.company.com/infra/pgbouncer:1.21
```

Portainer recognizes the registry URL prefix and uses the matching stored credentials.

## Step 7: Test Registry Connectivity

```bash
# Test authentication from the Docker host
docker login registry.company.com

# Test image pull
docker pull registry.company.com/team/myapp:latest

# List registry contents (if registry supports it)
curl -u registryuser:strongpassword \
  https://registry.company.com/v2/_catalog
```

## Step 8: Configure Registry Mirrors

For caching public images through your private registry, configure Docker to use it as a mirror:

```json
// /etc/docker/daemon.json
{
  "registry-mirrors": ["https://registry.company.com"]
}
```

Or configure a pull-through cache in your Harbor or Nexus instance.

## Troubleshooting

### Connection Refused

```bash
# Check registry is accessible
curl -I https://registry.company.com/v2/
# Expected: HTTP 401 Unauthorized (registry is up, auth required)
```

### TLS Certificate Errors

```bash
# Check cert validity
openssl s_client -connect registry.company.com:443 -servername registry.company.com

# If self-signed, add to trusted CAs (see Step 4)
```

### Authentication Failed

```bash
# Test credentials directly
curl -u username:password https://registry.company.com/v2/_catalog
# Should return JSON list of repositories
```

## Conclusion

Adding a custom private registry to Portainer enables seamless pulling of internal or proprietary container images for all deployments. Configure TLS for security, store credentials in Portainer rather than in Compose files, and use your private registry as a single source of truth for all container images in your organization.
