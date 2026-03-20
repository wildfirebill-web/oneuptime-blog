# How to Set Up Azure ACI as an Environment in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Azure, ACI, Cloud, DevOps

Description: Learn how to configure Azure Container Instances as a managed environment in Portainer Business Edition to deploy and manage serverless containers in Azure.

## Introduction

Azure Container Instances (ACI) allows you to run containers in Azure without managing underlying infrastructure. Portainer Business Edition supports ACI as a first-class environment type, letting you deploy and manage Azure containers from the same Portainer interface you use for Docker and Kubernetes environments.

## Prerequisites

- Portainer Business Edition (BE)
- An active Azure subscription
- Azure CLI installed locally or access to Azure Cloud Shell
- An Azure AD (Entra ID) Application registered with appropriate permissions
- Admin access to Portainer

## Architecture Overview

Portainer connects to ACI using Azure's API via an Azure AD service principal. The service principal must have the `Contributor` role on a resource group where containers will be deployed.

## Step 1: Gather Azure Prerequisites

You need the following information from Azure:

```bash
# Install Azure CLI if not present

curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Log into Azure
az login

# Get your subscription ID
az account list --output table
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
echo "Subscription ID: $SUBSCRIPTION_ID"

# Get your tenant ID
TENANT_ID=$(az account show --query tenantId -o tsv)
echo "Tenant ID: $TENANT_ID"
```

## Step 2: Create a Resource Group for ACI

```bash
# Create a dedicated resource group for Portainer-managed ACI containers
az group create \
  --name portainer-aci-rg \
  --location eastus

# Verify creation
az group show --name portainer-aci-rg
```

## Step 3: Register an Azure AD Application

```bash
# Create an Azure AD app registration
APP=$(az ad app create \
  --display-name "Portainer ACI Integration" \
  --query '{appId: appId, objectId: id}' -o json)

APP_ID=$(echo $APP | jq -r '.appId')
echo "Application (Client) ID: $APP_ID"
```

## Step 4: Create a Service Principal and Assign Role

```bash
# Create a service principal for the app
SP=$(az ad sp create --id $APP_ID --query 'id' -o tsv)

# Assign Contributor role on the resource group
az role assignment create \
  --assignee $APP_ID \
  --role "Contributor" \
  --scope "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/portainer-aci-rg"

echo "Service principal created and role assigned."
```

## Step 5: Create a Client Secret

```bash
# Create a client secret (valid for 1 year)
SECRET=$(az ad app credential reset \
  --id $APP_ID \
  --years 1 \
  --query 'password' -o tsv)

echo "Client Secret: $SECRET"
# Save this securely - it will not be shown again
```

## Step 6: Add ACI as an Environment in Portainer

1. Log into Portainer as admin.
2. Go to **Environments** → **Add environment**.
3. Select **Azure ACI** as the environment type.
4. Fill in:
   - **Name**: `Azure Production ACI`
   - **Subscription ID**: Your Azure subscription ID
   - **Tenant ID**: Your Azure AD tenant ID
   - **Application ID (Client ID)**: The app ID from step 3
   - **Authentication key (Client Secret)**: The secret from step 5
5. Click **Connect** to validate credentials.
6. If successful, click **Save environment**.

## Step 7: Verify the ACI Environment

After saving:

1. The ACI environment will appear in the Portainer home dashboard.
2. Click on it to open the ACI management view.
3. You should see an empty container list (ready for deployments).
4. The available Azure regions and resource groups will be populated.

## Configuring Environment Groups and Tags

Optionally, organize your ACI environment:

1. In environment settings, add **tags** like `cloud: azure`, `region: eastus`.
2. Assign the environment to an **environment group** (e.g., `cloud-environments`).
3. Set **team access** to restrict which Portainer users can deploy to this ACI environment.

## Permissions Summary

The Azure AD application needs at minimum:

| Permission | Scope | Purpose |
|-----------|-------|---------|
| `Contributor` | Resource Group | Create/delete ACI container groups |
| `Reader` | Subscription | List available regions and resource groups |

## Troubleshooting

```bash
# Test Azure credentials manually
az login --service-principal \
  --username $APP_ID \
  --password $SECRET \
  --tenant $TENANT_ID

# List ACI container groups to verify access
az container list --resource-group portainer-aci-rg --output table

# Check role assignments
az role assignment list --assignee $APP_ID --output table
```

## Conclusion

Setting up Azure ACI as a Portainer environment enables you to manage serverless Azure containers from the same dashboard as your Docker and Kubernetes workloads. The integration uses an Azure AD service principal with scoped Contributor access, providing a secure and auditable connection. Once configured, you can deploy containers to ACI with the same familiar Portainer workflow.
