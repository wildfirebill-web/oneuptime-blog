# How to Use Ceph Config from Kubernetes Secrets in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Secret, Configuration, Security

Description: Store sensitive Ceph configuration values in Kubernetes Secrets and reference them in Rook to avoid exposing credentials or keys directly in the CephCluster CRD.

---

## Why Store Ceph Config in Secrets

Some Ceph configuration values are sensitive - S3 access keys, LDAP bind credentials, or TLS certificates. Storing them directly in the `CephCluster` spec exposes them to anyone who can read the CRD. Kubernetes Secrets, while not encrypted by default, have RBAC controls and audit logging that provide a stronger access boundary.

Rook supports referencing Secrets for values used in the CSI config map and for custom Ceph configuration parameters.

## Storing Config Values in a Secret

Create a Kubernetes Secret containing Ceph configuration key-value pairs:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-config-overrides
  namespace: rook-ceph
type: Opaque
stringData:
  rgw_s3_auth_use_ldap: "true"
  rgw_ldap_uri: "ldap://ldap.example.com"
  rgw_ldap_binddn: "cn=admin,dc=example,dc=com"
  rgw_ldap_bindpw: "super-secret-password"
  rgw_ldap_searchdn: "ou=users,dc=example,dc=com"
```

## Referencing the Secret in CephObjectStore

For RGW (RADOS Gateway / S3) configuration, reference secrets via the `config` map in the `CephObjectStore`:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  gateway:
    port: 80
    instances: 1
  security:
    kms:
      connectionDetails:
        KMS_PROVIDER: "vault"
        VAULT_ADDR: "https://vault.example.com:8200"
      tokenSecretName: vault-kms-token
```

## Injecting Ceph Config from a Secret into Pods

For RGW config that must be injected at pod startup, use environment variables from secrets in the operator ConfigMap approach:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-ceph-operator-config
  namespace: rook-ceph
data:
  ROOK_CEPH_SECRET_VOLUME_MOUNTS: "ceph-custom-config:/etc/ceph/custom"
```

Then create a Secret with the custom ceph.conf-format content:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-custom-config
  namespace: rook-ceph
type: Opaque
stringData:
  custom.conf: |
    [global]
    auth_client_required = cephx
    [client.rgw.my-store]
    rgw_enable_usage_log = true
    rgw_usage_max_shards = 32
```

## Managing the Ceph Keyring Secret

Rook automatically creates and manages the Ceph admin keyring in a Secret:

```bash
kubectl -n rook-ceph get secret rook-ceph-admin-keyring -o yaml
```

If you need to rotate the admin key, update the Secret and the operator will synchronize it:

```bash
NEW_KEY=$(ceph auth get-or-create client.admin \
  mon 'profile rbd' osd 'profile rbd' mgr 'profile rbd' \
  | grep key | awk '{print $3}')
kubectl -n rook-ceph patch secret rook-ceph-admin-keyring \
  --type='json' -p="[{\"op\":\"replace\",\"path\":\"/data/keyring\",\"value\":\"$(echo -n "[client.admin]\n\tkey = ${NEW_KEY}" | base64 -w0)\"}]"
```

## Verifying Secret-Based Config

Confirm the Secret content is correctly read by describing the associated pod and checking mounted volumes:

```bash
kubectl -n rook-ceph describe pod -l app=rook-ceph-rgw | grep -A 10 "Volumes"
```

For config applied at the Ceph database level, verify through the toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config dump | grep rgw
```

## Summary

Storing sensitive Ceph configuration in Kubernetes Secrets separates credentials from CRD definitions, enabling RBAC-controlled access and audit logging. Use Secrets for RGW LDAP credentials, KMS tokens, and custom ceph.conf content. Reference them through the CephObjectStore security section or operator ConfigMap volume mounts to keep sensitive values out of Git repositories and CRD manifests.
