# How to Set Up Azure Key Vault with Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Encryption, Azure, Security

Description: Integrate Azure Key Vault as a KMS backend for Rook-Ceph encrypted volumes using service principal or managed identity authentication.

---

## Overview

Azure Key Vault provides centralized secret and key management for workloads running in Azure Kubernetes Service (AKS) or any Kubernetes cluster with Azure connectivity. Rook-Ceph supports Azure Key Vault as a KMS backend, enabling envelope encryption where per-volume keys are wrapped using Azure Key Vault keys.

## Prerequisites

- Azure subscription with Key Vault service
- Kubernetes cluster with network access to Azure Key Vault
- Azure Service Principal or Managed Identity with Key Vault access
- Rook-Ceph 1.12 or later

## Step 1 - Create an Azure Key Vault

```bash
az group create --name rook-ceph-rg --location eastus
az keyvault create \
  --name rook-ceph-kv \
  --resource-group rook-ceph-rg \
  --location eastus \
  --sku premium
```

## Step 2 - Create a Key in Key Vault

```bash
az keyvault key create \
  --vault-name rook-ceph-kv \
  --name rook-ceph-root-key \
  --protection hsm \
  --kty RSA-HSM \
  --size 4096
```

## Step 3 - Create a Service Principal with Key Vault Access

```bash
# Create service principal
az ad sp create-for-rbac \
  --name rook-ceph-kv-sp \
  --skip-assignment true

# Grant Key Vault access
az keyvault set-policy \
  --name rook-ceph-kv \
  --spn <service-principal-id> \
  --key-permissions get wrapKey unwrapKey create
```

## Step 4 - Store Azure Credentials as Kubernetes Secrets

```bash
kubectl create secret generic azure-kv-credentials \
  --from-literal=CLIENT_ID="<service-principal-id>" \
  --from-literal=CLIENT_SECRET="<service-principal-secret>" \
  --from-literal=TENANT_ID="<azure-tenant-id>" \
  -n rook-ceph
```

## Step 5 - Configure the KMS ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-ceph-csi-kms-config
  namespace: rook-ceph
data:
  config.json: |-
    {
      "azure-kv-kms": {
        "encryptionKMSType": "azure-kv",
        "AZURE_VAULT_URL": "https://rook-ceph-kv.vault.azure.net/",
        "AZURE_VAULT_KEY_NAME": "rook-ceph-root-key",
        "AZURE_VAULT_KEY_VERSION": "",
        "AZURE_CLIENT_ID": "azure-kv-credentials",
        "AZURE_CLIENT_SECRET": "azure-kv-credentials",
        "AZURE_TENANT_ID": "azure-kv-credentials"
      }
    }
```

## Step 6 - Create an Encrypted StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-azure-kv
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  encrypted: "true"
  encryptionKMSID: azure-kv-kms
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
```

## Workload Identity (AKS Recommended)

For AKS clusters, use Workload Identity instead of service principal secrets:

```bash
# Enable workload identity on AKS
az aks update --resource-group rook-rg --name rook-aks \
  --enable-workload-identity --enable-oidc-issuer

# Create a managed identity and federate it
az identity create --name rook-ceph-identity --resource-group rook-rg
az keyvault set-policy --name rook-ceph-kv \
  --object-id <managed-identity-principal-id> \
  --key-permissions get wrapKey unwrapKey create
```

## Summary

Azure Key Vault integration in Rook-Ceph provides HSM-backed encryption key management for workloads on Azure. Using envelope encryption, per-volume LUKS keys are wrapped with an Azure Key Vault key, ensuring the root key material never leaves the Azure HSM. For AKS environments, Workload Identity eliminates service principal credential management entirely.
