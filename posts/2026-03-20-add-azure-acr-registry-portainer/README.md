# How to Add Azure ACR as a Registry in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Azure, ACR, Container Registry, DevOps

Description: Learn how to connect Azure Container Registry (ACR) to Portainer to pull and manage private container images.

## Overview

Azure Container Registry (ACR) is Microsoft's managed Docker registry service. You can authenticate using admin credentials or service principals. Portainer supports ACR as a custom registry.

## Getting ACR Credentials

### Option 1: Admin Account (Simple, not recommended for production)

```bash
# Enable admin account on your ACR
az acr update -n myregistry --admin-enabled true

# Get admin credentials
az acr credential show -n myregistry
# Returns username and two passwords you can use in Portainer
```

### Option 2: Service Principal (Recommended for production)

```bash
# Create a service principal with pull-only access to your ACR
ACR_REGISTRY_ID=$(az acr show --name myregistry --query id --output tsv)

SP_PASSWD=$(az ad sp create-for-rbac \
  --name acr-service-principal \
  --scopes $ACR_REGISTRY_ID \
  --role acrpull \
  --query password --output tsv)

SP_APP_ID=$(az ad sp show \
  --id acr-service-principal \
  --query appId --output tsv)

echo "Username: ${SP_APP_ID}"
echo "Password: ${SP_PASSWD}"
```

## Adding ACR to Portainer

1. Go to **Settings > Registries** and click **Add registry**.
2. Select **Azure** as the registry type, or use **Custom registry**.
3. Fill in:
   - **Registry URL**: `myregistry.azurecr.io`
   - **Username**: Service principal app ID (or admin username)
   - **Password**: Service principal password (or admin password)
4. Click **Add registry**.

## Testing ACR Access from CLI

```bash
# Log in to ACR using service principal credentials
docker login myregistry.azurecr.io \
  --username <app-id> \
  --password <password>

# Pull an image to confirm access
docker pull myregistry.azurecr.io/my-app:latest
```

## Using ACR in a Stack Compose File

```yaml
version: "3.8"

services:
  app:
    # Portainer will use the stored ACR credentials to pull this image
    image: myregistry.azurecr.io/my-app:1.0.0
    deploy:
      replicas: 2
```

## Assigning ACR to Environments

After adding the registry, navigate to **Environments**, select your environment, and enable the ACR registry so containers in that environment can pull images from it.

## Conclusion

Azure Container Registry integrates cleanly with Portainer as a custom registry. For production setups, use a service principal with `acrpull` role rather than the admin account to follow the principle of least privilege.
