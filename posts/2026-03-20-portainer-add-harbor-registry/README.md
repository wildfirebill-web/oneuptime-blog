# How to Add a Harbor Registry to Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Harbor, Registry, Security, DevOps

Description: Learn how to add a Harbor container registry to Portainer for secure, enterprise-grade private image management.

## Introduction

Harbor is an open-source cloud-native registry that provides security, identity, and management features beyond what the basic Docker Registry offers. It includes vulnerability scanning, image signing, RBAC, and replication policies. Portainer has native support for Harbor, providing a tighter integration than generic custom registries. This guide covers adding Harbor to Portainer.

## Prerequisites

- Portainer CE or BE installed
- Harbor v2.x running and accessible
- Harbor project with images
- A Harbor robot account or user account

## Harbor Registry URL Format

```text
{harbor-domain}/{project}/{repository}:{tag}
# Examples:

harbor.company.com/myproject/myapp:latest
harbor.company.com/team/api:v2.0
```

## Step 1: Create a Harbor Robot Account

Robot accounts are the recommended way to give Portainer access to Harbor - they are dedicated to automation and have controlled scopes:

1. Log in to your Harbor instance as admin
2. Go to **Administration → Robot Accounts**
3. Click **+ New Robot Account**
4. Configure:
   ```text
   Name:         portainer-pull
   Duration:     365 (days)
   Description:  Portainer deployment access
   Level:        System level (for all projects) or Project level (specific projects)
   ```
5. Under **Permissions**, grant:
   - `Repository:Pull` (read access to all repositories)
   - `Artifact:Read` (read artifact metadata)
   - `Tag:List` (if Portainer needs to list tags)
6. Click **Add**
7. Copy the **Name** (format: `robot$portainer-pull`) and **Secret**

## Step 2: Add Harbor Registry in Portainer

1. Go to **Registries** in Portainer
2. Click **+ Add registry**
3. Select **Harbor**

## Step 3: Configure Harbor Connection

```text
Name:     Harbor Registry
URL:      https://harbor.company.com
Username: robot$portainer-pull      (include the "robot$" prefix)
Password: [robot account secret]
```

**Note:** Harbor robot account usernames always start with `robot$`.

4. Click **Add registry**

## Step 4: Verify Harbor Connectivity

Portainer validates the connection when saving. If it fails:

```bash
# Test Harbor API connectivity
curl -u 'robot$portainer-pull:secret' \
  https://harbor.company.com/api/v2.0/projects

# Test image pull
docker login harbor.company.com \
  -u 'robot$portainer-pull' \
  -p 'your-robot-secret'

docker pull harbor.company.com/myproject/myapp:latest
```

## Step 5: Use Harbor Images in Portainer Stacks

```yaml
version: "3.8"

services:
  app:
    image: harbor.company.com/production/myapp:v2.1.0
    # Portainer uses stored Harbor robot account credentials

  database-proxy:
    image: harbor.company.com/infrastructure/pgbouncer:latest
```

## Step 6: Browse Harbor Registry in Portainer BE

Portainer Business Edition allows browsing registry contents directly:

1. Go to **Registries**
2. Click on the Harbor registry
3. Click **Browse** (BE feature)
4. Navigate through projects → repositories → tags

This lets you select images directly without typing full image paths.

## Step 7: Configure Per-Project Robot Accounts

For fine-grained access control, create a separate robot account per project:

```bash
# Project-level robot accounts limit access to specific Harbor projects
# Creates: robot$portainer-prod+pull (project-scoped)
```

In Harbor:
1. Navigate to **Project → {project-name} → Robot Accounts**
2. Create a project-specific robot account
3. This account only has access to that project's repositories

## Step 8: Harbor Content Trust (Image Signing)

Harbor integrates with Notary for image signing. When content trust is enabled:

```bash
# Enable content trust on the Docker client
export DOCKER_CONTENT_TRUST=1
export DOCKER_CONTENT_TRUST_SERVER=https://harbor.company.com:4443

# Pull only signed images
docker pull harbor.company.com/myproject/myapp:latest
```

For Portainer to enforce content trust, configure the Docker daemon:

```json
// /etc/docker/daemon.json
{
  "content-trust": {
    "mode": "enforced",
    "trust-pinning": {
      "official-library-images": true
    }
  }
}
```

## Step 9: Harbor Vulnerability Scanning

Harbor can scan images for vulnerabilities automatically. Configure scan policies in Harbor:

1. **Harbor → Interrogation Services → Scanners** - Add Trivy or Clair
2. **Harbor → Projects → {project} → Configuration → Vulnerability scanning** - Enable
3. Configure vulnerability policy to prevent pulling images with critical CVEs

```bash
# Check vulnerability scan status via API
curl -u 'robot$portainer-pull:secret' \
  "https://harbor.company.com/api/v2.0/projects/myproject/repositories/myapp/artifacts/latest/scan"
```

## Troubleshooting

### Robot Account Access Denied

```text
Error: unauthorized: unauthorized to access repository
```

- Verify the robot account name includes `robot$` prefix
- Check that the robot account has `Repository:Pull` permission
- Ensure the account has not expired

### Certificate Error

```text
Error: x509: certificate signed by unknown authority
```

Add Harbor's CA certificate to the Docker host:

```bash
sudo mkdir -p /etc/docker/certs.d/harbor.company.com
sudo cp harbor-ca.crt /etc/docker/certs.d/harbor.company.com/ca.crt
sudo systemctl restart docker
```

## Conclusion

Harbor provides enterprise-grade features on top of the basic Docker registry, and Portainer's native Harbor support makes integration seamless. Use robot accounts with minimal permissions for deployment access, leverage Harbor's vulnerability scanning to ensure you only deploy secure images, and use Portainer BE's registry browsing feature to navigate Harbor's project structure directly from the Portainer interface.
