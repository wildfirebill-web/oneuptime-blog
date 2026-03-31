# How to Configure KMS Connection Details in Rook CephCluster CRD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Encryption, KMS, Kubernetes

Description: Configure the security.kms section of the Rook CephCluster CRD to enable OSD encryption with external key management services.

---

## Overview

The `security.kms` section of the `CephCluster` CRD controls OSD-level encryption using an external Key Management Service. Unlike CSI volume encryption (which is per-StorageClass), OSD-level KMS integration encrypts the underlying OSD block devices. This section covers all supported connection detail fields and their usage across different KMS backends.

## CephCluster Security Section Structure

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  security:
    kms:
      connectionDetails:
        KMS_PROVIDER: <provider>
        # Provider-specific fields follow
      tokenSecretName: <secret-name>
```

## HashiCorp Vault Connection Details

```yaml
spec:
  security:
    kms:
      connectionDetails:
        KMS_PROVIDER: vault
        VAULT_ADDR: https://vault.example.com:8200
        VAULT_BACKEND_PATH: secret/
        VAULT_BACKEND_KEY: luksKey
        VAULT_AUTH_METHOD: kubernetes
        VAULT_AUTH_KUBERNETES_ROLE: rook-ceph-kms
        VAULT_TLS_SERVER_NAME: vault.example.com
        VAULT_CACERT: vault-ca-cert
        VAULT_CLIENT_CERT: vault-client-cert
        VAULT_CLIENT_KEY: vault-client-key
```

## IBM Key Protect Connection Details

```yaml
spec:
  security:
    kms:
      connectionDetails:
        KMS_PROVIDER: ibmkeyprotect
        IBM_KP_SERVICE_INSTANCE_ID: <instance-id>
        IBM_KP_BASE_URL: https://us-south.kms.cloud.ibm.com
        IBM_KP_TOKEN_URL: https://iam.cloud.ibm.com/identity/token
        IBM_KP_REGION: us-south
      tokenSecretName: ibm-kp-credentials
```

## Azure Key Vault Connection Details

```yaml
spec:
  security:
    kms:
      connectionDetails:
        KMS_PROVIDER: azure-kv
        AZURE_VAULT_URL: https://myvault.vault.azure.net/
        AZURE_VAULT_KEY_NAME: rook-root-key
        AZURE_CLIENT_ID: azure-credentials
        AZURE_CLIENT_SECRET: azure-credentials
        AZURE_TENANT_ID: azure-credentials
```

## Secrets Metadata Connection Details

For development or simple deployments using Kubernetes Secrets:

```yaml
spec:
  security:
    kms:
      connectionDetails:
        KMS_PROVIDER: secrets-metadata
```

No `tokenSecretName` is needed for the secrets-metadata provider.

## Apply and Verify the Configuration

After updating the CephCluster CRD, verify the operator picks up the changes:

```bash
kubectl apply -f cephcluster.yaml
kubectl get cephcluster rook-ceph -n rook-ceph -o jsonpath='{.status.state}'
```

Watch the operator logs for KMS validation:

```bash
kubectl logs -n rook-ceph -l app=rook-ceph-operator --follow | grep -i kms
```

## Validate Connection Details Before Applying

Use the Rook toolbox to test KMS connectivity before applying to production:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config-key get dm-crypt/osd/0/luks/key
```

If the key retrieval fails, recheck the `connectionDetails` fields and the token secret content.

## Required vs Optional Fields

```text
Required fields (all providers):
  KMS_PROVIDER - the KMS type identifier

Optional/provider-specific fields:
  VAULT_*      - Only for Vault provider
  IBM_KP_*     - Only for IBM Key Protect
  AZURE_*      - Only for Azure Key Vault
  KMIP_*       - Only for KMIP-compliant servers
```

## Summary

The `security.kms` section of the CephCluster CRD is the single place where OSD-level KMS integration is configured. Each KMS provider has specific connection detail fields that map to environment variables consumed by the Ceph MGR. Correctly populating these fields along with the `tokenSecretName` for credential lookup enables automatic key management for all encrypted OSDs in the cluster.
