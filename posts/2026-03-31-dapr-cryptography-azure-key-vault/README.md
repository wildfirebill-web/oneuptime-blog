# How to Use Dapr Cryptography with Azure Key Vault

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Cryptography, Azure Key Vault, Encryption, Cloud, Security, Key Management

Description: Learn how to configure Dapr Cryptography with Azure Key Vault as the key provider, enabling cloud-managed key storage and HSM-backed encryption for production workloads.

---

## Why Use Azure Key Vault with Dapr Cryptography?

Azure Key Vault provides HSM-backed key storage, automatic key rotation, access policies, and audit logging. By using Azure Key Vault as the Dapr Cryptography key provider, your encryption keys never touch your application servers - Dapr retrieves them from Key Vault at runtime and performs all cryptographic operations inside the sidecar.

## Prerequisites

- An Azure Key Vault with a key (RSA or AES)
- Azure Managed Identity or a Service Principal for authentication
- Dapr installed in your environment

## Creating a Key in Azure Key Vault

```bash
# Create a Key Vault
az keyvault create --name my-app-keyvault \
  --resource-group myRG \
  --location eastus \
  --sku premium

# Create an RSA key for encryption
az keyvault key create \
  --vault-name my-app-keyvault \
  --name my-encryption-key \
  --kty RSA \
  --size 4096 \
  --ops encrypt decrypt wrapKey unwrapKey

# Grant the app's managed identity access
az keyvault set-policy \
  --name my-app-keyvault \
  --object-id <managed-identity-object-id> \
  --key-permissions get encrypt decrypt wrapKey unwrapKey
```

## Dapr Component Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azurekeyvault-crypto
spec:
  type: crypto.azure.keyvault
  version: v1
  metadata:
  - name: vaultURI
    value: "https://my-app-keyvault.vault.azure.net"
  # Using Managed Identity (recommended in AKS)
  - name: azureClientId
    value: "<managed-identity-client-id>"
```

For Service Principal authentication:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azurekeyvault-crypto
spec:
  type: crypto.azure.keyvault
  version: v1
  metadata:
  - name: vaultURI
    value: "https://my-app-keyvault.vault.azure.net"
  - name: azureTenantId
    value: "<tenant-id>"
  - name: azureClientId
    value: "<client-id>"
  - name: azureClientSecret
    secretKeyRef:
      name: azure-credentials
      key: clientSecret
```

## Encrypting Data with Azure Key Vault Key

```python
import io
from dapr.clients import DaprClient

def encrypt_sensitive_data(data: bytes, key_name: str = "my-encryption-key") -> bytes:
    with DaprClient() as d:
        encrypted = d.encrypt(
            data=io.BytesIO(data),
            options={
                "componentName": "azurekeyvault-crypto",
                "keyName": key_name,
                "keyWrapAlgorithm": "RSA-OAEP-256"
            }
        )
        return encrypted.read()

# Example: encrypt a database field
customer_ssn = "123-45-6789".encode()
encrypted_ssn = encrypt_sensitive_data(customer_ssn)
```

## Decrypting with Azure Key Vault

```python
def decrypt_sensitive_data(ciphertext: bytes, key_name: str = "my-encryption-key") -> bytes:
    with DaprClient() as d:
        decrypted = d.decrypt(
            data=io.BytesIO(ciphertext),
            options={
                "componentName": "azurekeyvault-crypto",
                "keyName": key_name
            }
        )
        return decrypted.read()

plaintext_ssn = decrypt_sensitive_data(encrypted_ssn).decode()
```

## AKS Pod Identity Setup

For seamless integration in AKS, use Workload Identity:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    metadata:
      labels:
        azure.workload.identity/use: "true"
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "my-app"
    spec:
      serviceAccountName: my-app-sa  # SA bound to managed identity
```

```bash
# Create service account with workload identity binding
az aks pod-identity add \
  --resource-group myRG \
  --cluster-name myAKS \
  --namespace default \
  --name my-app-identity \
  --identity-resource-id /subscriptions/.../resourceGroups/myRG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/my-app-mi
```

## Verifying Key Vault Access

```bash
# Test that the Dapr sidecar can reach Key Vault
kubectl exec -it <pod> -c daprd -- \
  curl -X POST http://localhost:3500/v1.0-alpha1/crypto/azurekeyvault-crypto/encrypt \
  -H "Content-Type: application/octet-stream" \
  -H "dapr-key-name: my-encryption-key" \
  -H "dapr-key-wrap-algorithm: RSA-OAEP-256" \
  --data "test data"
```

## Key Vault Audit Logging

Enable diagnostic settings to log all cryptographic operations:

```bash
az monitor diagnostic-settings create \
  --name keyvault-audit \
  --resource /subscriptions/.../resourceGroups/myRG/providers/Microsoft.KeyVault/vaults/my-app-keyvault \
  --logs '[{"category": "AuditEvent", "enabled": true}]' \
  --workspace <log-analytics-workspace-id>
```

## Summary

Dapr Cryptography with Azure Key Vault provides production-grade key management with HSM-backed storage, access policies, and audit trails. By using Managed Identity in AKS, your application authenticates to Key Vault without credentials in code or environment variables, and Dapr handles all key retrieval and cryptographic operations transparently.
