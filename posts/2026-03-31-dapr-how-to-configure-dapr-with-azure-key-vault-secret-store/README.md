# How to Configure Dapr with Azure Key Vault Secret Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure Key Vault, Secret, Security, Microservice

Description: Learn how to configure Dapr to use Azure Key Vault as a secret store, enabling your microservices to securely access secrets without hardcoding credentials.

---

## Why Use Azure Key Vault with Dapr

Dapr's secret store API abstracts how secrets are retrieved, letting your application fetch secrets through the Dapr sidecar regardless of where they are stored. Azure Key Vault is a managed secret store that integrates naturally with Azure-hosted workloads and provides audit logging, rotation, and access control.

## Prerequisites

- An Azure subscription
- An Azure Key Vault instance with secrets
- A Dapr-enabled application (self-hosted or AKS)
- An identity with Key Vault access (Managed Identity or Service Principal)

## Set Up Azure Key Vault

Create a Key Vault and add secrets:

```bash
# Create a resource group
az group create --name my-rg --location eastus

# Create a Key Vault
az keyvault create \
  --name my-dapr-vault \
  --resource-group my-rg \
  --location eastus

# Add secrets
az keyvault secret set \
  --vault-name my-dapr-vault \
  --name "db-connection-string" \
  --value "Server=mydb.postgres.database.azure.com;Database=myapp;..."

az keyvault secret set \
  --vault-name my-dapr-vault \
  --name "api-key" \
  --value "super-secret-api-key"
```

## Define the Azure Key Vault Secret Store Component

Using Managed Identity (recommended for AKS):

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azurekeyvault
  namespace: default
spec:
  type: secretstores.azure.keyvault
  version: v1
  metadata:
  - name: vaultName
    value: "my-dapr-vault"
  - name: azureEnvironment
    value: "AZUREPUBLICCLOUD"
```

Using Service Principal (for self-hosted or non-Managed Identity scenarios):

```yaml
  metadata:
  - name: vaultName
    value: "my-dapr-vault"
  - name: azureTenantId
    value: "your-tenant-id"
  - name: azureClientId
    value: "your-client-id"
  - name: azureClientSecret
    value: "your-client-secret"
```

## Grant Access to the Key Vault

For a Managed Identity (AKS workload identity):

```bash
# Get the managed identity's object ID
IDENTITY_OBJECT_ID=$(az aks show \
  --resource-group my-rg \
  --name my-cluster \
  --query identityProfile.kubeletidentity.objectId -o tsv)

# Grant Key Vault access
az keyvault set-policy \
  --name my-dapr-vault \
  --object-id $IDENTITY_OBJECT_ID \
  --secret-permissions get list
```

## Read Secrets in Your Application

Use the Dapr HTTP API to read secrets:

```bash
curl http://localhost:3500/v1.0/secrets/azurekeyvault/db-connection-string
```

Response:

```json
{
  "db-connection-string": "Server=mydb.postgres.database.azure.com;..."
}
```

## Use Secrets in Node.js

```javascript
const { DaprClient } = require('@dapr/dapr');

const client = new DaprClient();

async function getSecret(secretName) {
  const secret = await client.secret.get('azurekeyvault', secretName);
  return secret[secretName];
}

async function initializeApp() {
  const dbConnectionString = await getSecret('db-connection-string');
  const apiKey = await getSecret('api-key');

  console.log('Secrets loaded successfully');
  // Use secrets to initialize database connections, HTTP clients, etc.
  await connectToDatabase(dbConnectionString);
  initializeApiClient(apiKey);
}
```

## Use Secrets in Python

```python
from dapr.clients import DaprClient

def get_secret(secret_name: str) -> str:
    with DaprClient() as client:
        secret = client.get_secret(
            store_name='azurekeyvault',
            key=secret_name
        )
        return secret.secret[secret_name]

# Load configuration from Key Vault
db_url = get_secret('db-connection-string')
api_key = get_secret('api-key')
print("Configuration loaded from Azure Key Vault")
```

## Reference Key Vault Secrets in Other Components

Use the secret store in other Dapr component definitions:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: postgres-db
spec:
  type: bindings.postgresql
  version: v1
  metadata:
  - name: connectionString
    secretKeyRef:
      name: db-connection-string
      key: db-connection-string
auth:
  secretStore: azurekeyvault
```

## Restrict Secret Access with Namespaces

Limit which secrets your application can access:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  secrets:
    scopes:
    - storeName: azurekeyvault
      defaultAccess: deny
      allowedSecrets:
      - "db-connection-string"
      - "api-key"
```

## Summary

Configuring Dapr with Azure Key Vault as a secret store enables microservices to fetch secrets securely through the Dapr sidecar without hardcoded credentials in application code or component files. Managed Identity integration on AKS eliminates the need for long-lived credentials entirely, and Dapr's secret scoping provides fine-grained control over which secrets each application can access.
