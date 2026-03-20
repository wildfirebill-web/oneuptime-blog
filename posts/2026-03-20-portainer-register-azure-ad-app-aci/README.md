# How to Register an Azure AD Application for Portainer ACI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Azure, ACI, Azure AD, Security

Description: Learn how to register an Azure Active Directory application and configure its permissions to enable Portainer Business Edition to manage Azure Container Instances.

## Introduction

To connect Portainer to Azure Container Instances, you must first register an Azure AD (Microsoft Entra ID) application. This app acts as the identity that Portainer uses to authenticate with Azure APIs. This guide covers the full registration process with both the Azure Portal UI and Azure CLI.

## Prerequisites

- An active Azure subscription
- Azure AD Global Administrator or Application Administrator role
- Azure CLI installed (for CLI approach)
- Access to the Azure Portal

## Understanding the Azure AD App Registration

An Azure AD app registration creates an application identity that:
- Has its own `Application (Client) ID`
- Belongs to an Azure AD tenant (identified by `Tenant ID`)
- Can be granted roles on Azure resources
- Uses client secrets or certificates to authenticate

Portainer uses these credentials to make API calls to create and manage ACI container groups on your behalf.

## Method 1: Register via Azure Portal

### Step 1: Open App Registrations

1. Log into the [Azure Portal](https://portal.azure.com).
2. Search for **App registrations** in the top search bar.
3. Click **App registrations**.
4. Click **New registration**.

### Step 2: Fill in Registration Details

- **Name**: `Portainer ACI Integration`
- **Supported account types**: Select **Accounts in this organizational directory only (Single tenant)**
- **Redirect URI**: Leave blank (not required for service-to-service auth)

Click **Register**.

### Step 3: Record the IDs

After registration, on the app overview page, note:
- **Application (client) ID**: Copy this for Portainer configuration
- **Directory (tenant) ID**: Copy this for Portainer configuration

## Method 2: Register via Azure CLI

```bash
# Log into Azure

az login

# Create the app registration
APP=$(az ad app create \
  --display-name "Portainer ACI Integration" \
  --sign-in-audience AzureADMyOrg \
  --query '{displayName: displayName, appId: appId, id: id}' \
  -o json)

echo "Registration result:"
echo $APP | jq .

# Extract the Application ID
APP_ID=$(echo $APP | jq -r '.appId')
echo "Application (Client) ID: $APP_ID"

# Get the Tenant ID
TENANT_ID=$(az account show --query tenantId -o tsv)
echo "Tenant ID: $TENANT_ID"

# Get your Subscription ID
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
echo "Subscription ID: $SUBSCRIPTION_ID"
```

## Step 4: Create a Service Principal

The service principal is the representation of your app in your specific Azure AD tenant. It's what receives role assignments:

```bash
# Create the service principal
SP_ID=$(az ad sp create \
  --id $APP_ID \
  --query 'id' -o tsv)

echo "Service Principal Object ID: $SP_ID"
```

## Step 5: Assign the Contributor Role

Grant the app permission to manage resources in your ACI resource group:

```bash
# Create resource group if it doesn't exist
az group create --name portainer-aci-rg --location eastus

# Assign Contributor role on the resource group
az role assignment create \
  --assignee $APP_ID \
  --role "Contributor" \
  --scope "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/portainer-aci-rg"

# Verify the assignment
az role assignment list \
  --assignee $APP_ID \
  --scope "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/portainer-aci-rg" \
  --output table
```

## Step 6: Verify the Registration

Confirm the app is properly configured:

```bash
# Show app details
az ad app show --id $APP_ID --query '{
  displayName: displayName,
  appId: appId,
  signInAudience: signInAudience
}' -o json

# Verify the service principal
az ad sp show --id $APP_ID --query '{
  displayName: displayName,
  appId: appId,
  accountEnabled: accountEnabled
}' -o json
```

## Required API Permissions

For standard ACI management, the `Contributor` role assignment at the resource group level is sufficient. You do NOT need to configure API permissions in the Azure Portal's "API permissions" section for this use case, as Portainer uses Azure Resource Manager APIs which are controlled by Azure RBAC (role assignments), not OAuth scopes.

## Summary of Values for Portainer

After completing registration, you have:

```bash
echo "=== Portainer ACI Configuration Values ==="
echo "Subscription ID: $SUBSCRIPTION_ID"
echo "Tenant ID:       $TENANT_ID"
echo "Application ID:  $APP_ID"
echo "Resource Group:  portainer-aci-rg"
echo ""
echo "Next step: Create a client secret (see next guide)"
```

## Security Best Practices

- Use a **dedicated app registration** per environment (dev, staging, prod)
- Apply the **Principle of Least Privilege**: scope the Contributor role to the specific resource group, not the entire subscription
- Rotate client secrets on a schedule (every 6-12 months)
- Monitor sign-in logs in Azure AD for anomalous activity

## Conclusion

Registering an Azure AD application for Portainer ACI is a straightforward process that establishes the identity Portainer uses to manage your Azure containers. Scope the role assignment to the specific resource group Portainer will use, record the Application ID and Tenant ID, and proceed to create a client secret to complete the Portainer integration.
