# How to Set Up OSD Encryption with KMS on PVC Clusters in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Encryption, KMS, Kubernetes

Description: Configure OSD at-rest encryption backed by an external KMS for PVC-based Rook-Ceph clusters to protect data on dynamically provisioned storage.

---

## Overview

For Rook clusters using PVC-based OSD storage (common in cloud environments), enabling encryption with an external KMS ensures data at rest is protected by keys managed outside the cluster. This is required for compliance scenarios where key material must not reside on the same system as encrypted data.

## Architecture

In a PVC-based encrypted OSD setup:
1. Each OSD PVC contains a LUKS-encrypted block device
2. LUKS passphrase is stored in an external KMS (Vault, IBM Key Protect, etc.)
3. On OSD startup, Rook CSI retrieves the key from KMS and unlocks the device
4. On OSD shutdown, the key is not stored locally

## Configure the KMS ConfigMap

First, set up the `rook-ceph-csi-kms-config` ConfigMap with your KMS details. For HashiCorp Vault:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-ceph-csi-kms-config
  namespace: rook-ceph
data:
  config.json: |-
    {
      "vault-osd-kms": {
        "encryptionKMSType": "vault",
        "vaultAddress": "https://vault.example.com:8200",
        "vaultAuthPath": "/v1/auth/kubernetes/login",
        "vaultRole": "rook-osd-encryption",
        "vaultPassphraseRoot": "/v1/secret",
        "vaultPassphrasePath": "rook-ceph/osd/"
      }
    }
```

## Enable Encryption in CephCluster CRD

Configure the OSD `storageDeviceSets` with encryption enabled:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    storageClassDeviceSets:
    - name: set1
      count: 3
      encrypted: true
      portable: true
      volumeClaimTemplates:
      - metadata:
          name: data
        spec:
          resources:
            requests:
              storage: 100Gi
          storageClassName: local-storage
          volumeMode: Block
          accessModes:
            - ReadWriteOnce
  security:
    kms:
      connectionDetails:
        KMS_PROVIDER: vault
        VAULT_ADDR: https://vault.example.com:8200
        VAULT_BACKEND_PATH: secret
        VAULT_AUTH_METHOD: kubernetes
        VAULT_AUTH_KUBERNETES_ROLE: rook-osd-encryption
      tokenSecretName: rook-vault-kms-token
```

## Create the Vault Token Secret

```bash
kubectl create secret generic rook-vault-kms-token \
  --from-literal=token="<vault-token>" \
  -n rook-ceph
```

## Verify Encryption is Active

After deployment, check that OSD pods initialized with LUKS encryption:

```bash
kubectl logs -n rook-ceph -l app=rook-ceph-osd --container osd | grep -i "luks\|encrypt"
```

From the toolbox, verify OSD encryption status:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph osd dump | grep -i encrypt
```

## Summary

OSD encryption with KMS on PVC-based Rook clusters provides LUKS block-level encryption for all stored data, with key material held exclusively in an external KMS. The `encrypted: true` flag on `storageClassDeviceSets`, combined with the KMS connection details in the CephCluster security spec, automates the entire key lifecycle from OSD provisioning through normal operation.
