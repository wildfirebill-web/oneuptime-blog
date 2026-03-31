# How to Manage Token Secrets for KMS in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, KMS, Encryption, Secret

Description: Learn how to manage token secrets for Key Management Service (KMS) integration in Rook-Ceph to enable at-rest encryption for OSD volumes.

---

## Overview

Rook-Ceph supports Key Management Service (KMS) integration for at-rest encryption of OSD data. KMS providers such as HashiCorp Vault require authentication tokens that must be stored as Kubernetes secrets and referenced in the CephCluster configuration.

## Creating the KMS Token Secret

For HashiCorp Vault, you need a token with read access to the encryption key path. Create the Kubernetes secret in the same namespace as the Rook operator (typically `rook-ceph`):

```bash
kubectl create secret generic rook-vault-kms-token \
  --from-literal=token=<your-vault-token> \
  -n rook-ceph
```

Verify the secret was created:

```bash
kubectl get secret rook-vault-kms-token -n rook-ceph -o yaml
```

## Configuring the CephCluster for KMS

Reference the secret in the `CephCluster` CR under the `security` section:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  security:
    kms:
      enable: true
      connectionDetails:
        KMS_PROVIDER: vault
        VAULT_ADDR: https://vault.example.com:8200
        VAULT_BACKEND_PATH: secret
        VAULT_SECRET_ENGINE: kv
        VAULT_AUTH_METHOD: token
      tokenSecretName: rook-vault-kms-token
```

The `tokenSecretName` field tells Rook which Kubernetes secret holds the Vault token.

## Rotating the KMS Token

When the Vault token expires or is rotated, update the Kubernetes secret with the new token value:

```bash
kubectl patch secret rook-vault-kms-token -n rook-ceph \
  --type=merge \
  -p '{"stringData":{"token":"<new-vault-token>"}}'
```

Rook reads the token at OSD startup, so existing running OSDs are not immediately affected. Pods scheduled or restarted after the update will use the new token.

## Using Vault Kubernetes Auth Instead of Tokens

For production environments, using Vault Kubernetes authentication is more secure than static tokens. Configure the Kubernetes auth method in Vault:

```bash
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host=https://kubernetes.default.svc \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

vault write auth/kubernetes/role/rook-ceph \
  bound_service_account_names=rook-ceph-osd \
  bound_service_account_namespaces=rook-ceph \
  policies=rook-policy \
  ttl=24h
```

Then update the `connectionDetails` to use `VAULT_AUTH_METHOD: kubernetes` and remove the `tokenSecretName`.

## Verifying KMS Connectivity

Check that Rook can communicate with the KMS by inspecting OSD pod logs:

```bash
kubectl logs -n rook-ceph -l app=rook-ceph-osd --container osd | grep -i kms
```

Look for successful key fetch messages. If errors appear, verify the Vault address is reachable from within the cluster and that the token has the correct policy permissions.

## Summary

Managing KMS token secrets for Rook involves creating a Kubernetes secret with the provider token, referencing it in the CephCluster CR, and rotating it when needed. For production deployments, prefer Vault Kubernetes auth over static tokens to eliminate secret rotation overhead and improve security posture.
