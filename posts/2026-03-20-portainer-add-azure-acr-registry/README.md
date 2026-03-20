# How to Add Azure ACR as a Registry in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Azure, ACR, Registry, DevOps

Description: Learn how to configure Azure Container Registry (ACR) as a registry in Portainer for pulling images from Microsoft Azure.

## Introduction

Azure Container Registry (ACR) is Microsoft's managed Docker registry service tightly integrated with Azure services. Portainer supports ACR authentication using service principals or admin credentials. This guide covers both methods for integrating ACR with Portainer.

## Prerequisites

- Portainer CE or BE installed
- An Azure subscription with an ACR instance
- Azure CLI installed (for setup steps)
- Admin access to Portainer

## Step 1: Identify Your ACR Login Server

Your ACR login server URL follows this format:

```text
{registry-name}.azurecr.io
```

```bash
# List all ACR instances in your subscription

az acr list --query '[].{name:name,loginServer:loginServer}' -o table

# Output:
# Name        LoginServer
# myregistry  myregistry.azurecr.io
```

## Step 2: Option A - Enable Admin User (Simple but Less Secure)

For testing or simple setups:

```bash
# Enable admin user on your ACR
az acr update --name myregistry --admin-enabled true

# Get admin credentials
az acr credential show --name myregistry
# Output:
# {
#   "username": "myregistry",
#   "passwords": [
#     {"name": "password",  "value": "xxxxx"},
#     {"name": "password2", "value": "yyyyy"}
#   ]
# }
```

Add to Portainer:
```text
Registry type:  Custom registry (or Azure ACR if available)
URL:           myregistry.azurecr.io
Username:      myregistry
Password:      xxxxx    (password or password2)
```

## Step 3: Option B - Service Principal (Recommended for Production)

Create a service principal with pull-only access:

```bash
# Variables
ACR_NAME="myregistry"
SERVICE_PRINCIPAL_NAME="portainer-acr-pull"

# Get ACR resource ID
ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --query id --output tsv)

# Create service principal with acrpull role
SP=$(az ad sp create-for-rbac \
  --name $SERVICE_PRINCIPAL_NAME \
  --role acrpull \
  --scopes $ACR_REGISTRY_ID \
  --years 2)

# Extract credentials
SP_ID=$(echo $SP | jq -r .appId)
SP_PASSWORD=$(echo $SP | jq -r .password)

echo "Service Principal ID: $SP_ID"
echo "Service Principal Password: $SP_PASSWORD"
```

Add to Portainer:

```text
Registry type:  Custom registry
URL:           myregistry.azurecr.io
Username:      {service-principal-id}    (the appId)
Password:      {service-principal-password}
```

## Step 4: Add ACR in Portainer

1. Go to **Registries** in Portainer
2. Click **+ Add registry**
3. Select **Custom registry** (or Azure ACR if your Portainer version has it)
4. Fill in:

```text
Name:       Azure ACR (myregistry)
URL:        myregistry.azurecr.io
Username:   {admin-username or SP app ID}
Password:   {admin-password or SP password}
```

5. Click **Add registry**

## Step 5: Verify ACR Connectivity

```bash
# Test login with service principal
docker login myregistry.azurecr.io \
  --username $SP_ID \
  --password $SP_PASSWORD

# Pull an image
docker pull myregistry.azurecr.io/myimage:latest

# List repositories
az acr repository list --name myregistry --output table
```

## Step 6: Use ACR Images in Portainer Stacks

```yaml
version: "3.8"

services:
  app:
    image: myregistry.azurecr.io/myapp:latest
    # Portainer uses stored ACR credentials automatically

  api:
    image: myregistry.azurecr.io/myapi:v2.1.0
```

## Step 7: Push Images to ACR

```bash
# Tag local image for ACR
docker tag myapp:latest myregistry.azurecr.io/myapp:latest
docker tag myapp:latest myregistry.azurecr.io/myapp:v1.0.0

# Push to ACR
docker push myregistry.azurecr.io/myapp:latest
docker push myregistry.azurecr.io/myapp:v1.0.0

# Or use az acr build for cloud-side builds
az acr build \
  --registry myregistry \
  --image myapp:latest \
  .
```

## Step 8: Rotate Service Principal Credentials

Service principal credentials should be rotated periodically:

```bash
# Reset service principal password
az ad sp credential reset \
  --id $SP_ID \
  --years 2

# Update Portainer with the new password:
# Go to Registries → Edit the ACR registry → Update password
```

## Step 9: Use Managed Identity (Azure Kubernetes Service)

If Portainer runs on AKS with managed identity:

```bash
# Grant AKS kubelet identity pull access to ACR
az aks update \
  --name myakscluster \
  --resource-group myrg \
  --attach-acr myregistry

# AKS nodes can now pull from ACR without explicit credentials
```

## Geo-Replication (Multiple Regions)

If your ACR is geo-replicated, the same login server serves all regions:

```text
Primary:   myregistry.azurecr.io
Replicas:  Automatically created in: eastus, westeurope, etc.
```

All replicas use the same credentials - no additional Portainer configuration needed.

## Troubleshooting

### Unauthorized Error

```text
Error: unauthorized: authentication required
```

- Verify the service principal or admin credentials
- Check that the SP has `acrpull` role assignment
- Ensure the SP is not expired: `az ad sp show --id $SP_ID`

### Repository Not Found

```text
Error: repository name not valid
```

Use the full image path including the ACR login server prefix.

## Conclusion

Integrating Azure Container Registry with Portainer enables seamless private image pulls for your Azure-hosted images. For production environments, use a service principal with `acrpull` role instead of the admin account. Remember to rotate credentials periodically and update Portainer's registry configuration accordingly.
