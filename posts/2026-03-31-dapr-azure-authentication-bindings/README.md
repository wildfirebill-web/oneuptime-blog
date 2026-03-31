# How to Configure Azure Authentication for Dapr Bindings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure, Authentication, Binding, Security

Description: Learn the Azure authentication options for Dapr bindings, including connection strings, access keys, service principals, and Managed Identity for secure credential management.

---

## Azure Authentication Methods for Dapr Bindings

Dapr Azure bindings support several authentication mechanisms. The right choice depends on where your service runs and your security requirements. This guide covers each method with practical configuration examples.

## Method 1: Connection Strings

The simplest approach - uses the full Azure resource connection string:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: event-stream
spec:
  type: bindings.azure.eventhubs
  version: v1
  metadata:
    - name: connectionString
      secretKeyRef:
        name: azure-secrets
        key: eventhubsConnectionString
```

Always use Dapr secret references - never hardcode connection strings:

```bash
kubectl create secret generic azure-secrets \
  --from-literal=eventhubsConnectionString="Endpoint=sb://..."
```

## Method 2: Storage Account Access Keys

For Blob Storage and Storage Queues:

```yaml
  metadata:
    - name: storageAccount
      value: "mystorageaccount"
    - name: storageAccessKey
      secretKeyRef:
        name: azure-secrets
        key: storageAccessKey
```

Retrieve the key:

```bash
az storage account keys list \
  --account-name mystorageaccount \
  --resource-group my-rg \
  --query '[0].value' \
  --output tsv
```

## Method 3: Service Principal (Client Credentials)

For programmatic access with an Azure AD application:

```bash
# Create the service principal
az ad sp create-for-rbac \
  --name dapr-binding-sp \
  --role "Storage Blob Data Contributor" \
  --scopes /subscriptions/{sub}/resourceGroups/my-rg/providers/Microsoft.Storage/storageAccounts/mystorageaccount
```

This outputs `appId` (client ID), `password` (client secret), and `tenant`.

```yaml
  metadata:
    - name: storageAccount
      value: "mystorageaccount"
    - name: azureClientId
      secretKeyRef:
        name: azure-sp-secret
        key: clientId
    - name: azureClientSecret
      secretKeyRef:
        name: azure-sp-secret
        key: clientSecret
    - name: azureTenantId
      value: "your-tenant-id"
```

```bash
kubectl create secret generic azure-sp-secret \
  --from-literal=clientId=<appId> \
  --from-literal=clientSecret=<password>
```

## Method 4: Managed Identity (Recommended for AKS)

Azure Managed Identity is the most secure approach for AKS - no credentials to manage:

```bash
# Enable Workload Identity on AKS
az aks update \
  --name my-aks-cluster \
  --resource-group my-rg \
  --enable-oidc-issuer \
  --enable-workload-identity

# Create a User Assigned Managed Identity
az identity create \
  --name dapr-binding-identity \
  --resource-group my-rg

# Get the Client ID
CLIENT_ID=$(az identity show \
  --name dapr-binding-identity \
  --resource-group my-rg \
  --query clientId \
  --output tsv)

# Assign role to the identity
az role assignment create \
  --assignee $CLIENT_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{sub}/resourceGroups/my-rg/providers/Microsoft.Storage/storageAccounts/mystorageaccount
```

Federate with the Kubernetes service account:

```bash
OIDC_ISSUER=$(az aks show \
  --name my-aks-cluster \
  --resource-group my-rg \
  --query "oidcIssuerProfile.issuerUrl" \
  --output tsv)

az identity federated-credential create \
  --name dapr-binding-federated \
  --identity-name dapr-binding-identity \
  --resource-group my-rg \
  --issuer $OIDC_ISSUER \
  --subject "system:serviceaccount:default:order-service" \
  --audiences api://AzureADTokenExchange
```

Configure the binding with no credentials - just the Managed Identity client ID:

```yaml
  metadata:
    - name: storageAccount
      value: "mystorageaccount"
    - name: azureClientId
      value: "<managed-identity-client-id>"
```

## Kubernetes Service Account Annotation

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: default
  annotations:
    azure.workload.identity/client-id: "<managed-identity-client-id>"
```

## Choosing the Right Method

| Environment | Recommended Method |
|-------------|-------------------|
| Local development | Connection string via local file secret store |
| AKS | Workload Identity (Managed Identity) |
| Azure VMs / VMSS | System-assigned Managed Identity |
| CI/CD pipelines | Service Principal with OIDC |
| Cross-tenant | Service Principal |

## Summary

Azure authentication for Dapr bindings ranges from simple connection strings for development to Managed Identity with Workload Identity for production AKS deployments. Managed Identity is the recommended approach as it eliminates static credential management entirely. Always use Dapr secret references to avoid exposing credentials in component YAML files, regardless of which authentication method you choose.
