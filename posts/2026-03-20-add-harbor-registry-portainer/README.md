# How to Add a Harbor Registry to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Harbor, Container Registry, Security, DevOps

Description: Learn how to connect a Harbor container registry to Portainer for secure, enterprise-grade image management.

## What Is Harbor?

Harbor is an open-source, CNCF-graduated container registry that adds security features on top of a standard Docker registry, including vulnerability scanning, role-based access control (RBAC), image signing, and replication. It's widely used in enterprise and air-gapped environments.

## Prerequisites

- A running Harbor instance (e.g., at `https://harbor.mycompany.com`)
- A Harbor user account with read access to the target project

## Creating a Harbor Robot Account for Portainer

Robot accounts are service accounts in Harbor recommended for integrations:

1. In Harbor, go to your **Project > Robot Accounts**.
2. Click **Add Robot Account**.
3. Set the name (e.g., `portainer-puller`) and permissions (at minimum `Push` or `Pull` repository).
4. Copy the robot account name and token.

## Adding Harbor to Portainer

1. Go to **Settings > Registries** and click **Add registry**.
2. Select **Custom registry**.
3. Enter:
   - **Registry URL**: `harbor.mycompany.com`
   - **Username**: Robot account name (e.g., `robot$portainer-puller`)
   - **Password**: Robot account token
4. Click **Add registry**.

## Verifying Access via CLI

```bash
# Log in to Harbor via Docker CLI

docker login harbor.mycompany.com \
  -u "robot\$portainer-puller" \
  -p <robot-token>

# Pull an image from Harbor
docker pull harbor.mycompany.com/myproject/my-app:latest
```

## Using Harbor Images in a Stack

```yaml
version: "3.8"

services:
  app:
    # Portainer will use the stored Harbor credentials
    image: harbor.mycompany.com/myproject/my-app:1.5.0
    deploy:
      replicas: 3
```

## Harbor's Security Features Worth Enabling

- **Vulnerability scanning**: Enable auto-scan on push to detect CVEs before deployment.
- **Content trust**: Require image signing via Notary.
- **CVE allowlist**: Define acceptable vulnerabilities per project.
- **Deployment security**: Block deployment of images with critical vulnerabilities.

```bash
# Check image scan results via Harbor CLI/API
curl -u admin:harbor_password \
  "https://harbor.mycompany.com/api/v2.0/projects/myproject/repositories/my-app/artifacts/latest/vulnerabilities"
```

## Troubleshooting

- If you see `unauthorized: authentication required`, verify the robot account name starts with `robot$`.
- If Harbor uses a self-signed certificate, add it to Docker's trusted certificates or use `--insecure-registry`.

## Conclusion

Harbor is an excellent choice for organizations needing a feature-rich, self-hosted registry with security scanning. Integrating it with Portainer via robot accounts provides a clean separation of concerns and easy credential rotation.
