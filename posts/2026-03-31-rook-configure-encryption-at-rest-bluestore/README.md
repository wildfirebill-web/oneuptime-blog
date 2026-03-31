# How to Configure Encryption at Rest with BlueStore in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, BlueStore, Encryption, Security, Kubernetes

Description: Learn how to enable and configure BlueStore encryption at rest in Ceph to protect stored data on OSD disks from unauthorized physical access.

---

Ceph's BlueStore storage backend supports native encryption at rest. When enabled, all data written to an OSD is encrypted on disk using LUKS (Linux Unified Key Setup) with AES-XTS encryption. This protects data if physical storage media is removed or stolen.

## How BlueStore Encryption Works

BlueStore encryption operates at the OSD level. Each OSD generates a unique encryption key that is stored in the Ceph Monitor's key-value store (encrypted). When an OSD starts, it fetches its key from the Monitor and uses it to decrypt the data on disk.

This approach means:
- Data at rest is always encrypted
- Encryption keys are managed by the Monitor cluster
- No data is accessible without a running Monitor

## Enabling Encryption in Rook

Encryption must be enabled when creating OSDs - it cannot be added to existing OSDs without reprovisioning.

Configure encryption in the CephCluster resource:

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
                  storage: 10Gi
              storageClassName: local-storage
              volumeMode: Block
              accessModes:
                - ReadWriteOnce
```

The `encrypted: true` field enables BlueStore encryption for all OSDs in this device set.

## Verifying Encryption is Active

After the OSDs are provisioned, verify encryption:

```bash
# Check if an OSD is using encryption
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd metadata 0 | grep -i "encrypt\|bluefs"

# Look for the dmcrypt device
kubectl -n rook-ceph exec -it rook-ceph-osd-0-<hash> -- \
  lsblk | grep dm
```

The OSD metadata should show `osd_objectstore: bluestore` and reference dmcrypt devices.

## Using an External Key Management System

For stricter security, store encryption keys in HashiCorp Vault instead of the Monitor key-value store:

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
        KMS_PROVIDER: vault
        VAULT_ADDR: https://vault.example.com:8200
        VAULT_BACKEND: v2
        VAULT_SECRET_PATH: rook/osd-encryption
        VAULT_AUTH_METHOD: kubernetes
        VAULT_AUTH_KUBERNETES_ROLE: rook-ceph
      tokenSecretName: rook-vault-token
  storage:
    storageClassDeviceSets:
      - name: set1
        count: 3
        encrypted: true
```

## Backup and Key Recovery

Because encryption keys are stored in the Monitor, monitor backup is critical:

```bash
# Export Monitor key-value store (includes OSD encryption keys)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config-key dump > /tmp/ceph-config-keys-backup.json
```

Store this backup securely - loss of both the Monitor data and this backup means data is permanently unrecoverable.

## Performance Impact

BlueStore encryption using AES-XTS has minimal performance impact on modern hardware with AES-NI:

```bash
# Benchmark encrypted vs unencrypted (run on separate pools)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados bench -p encrypted-pool 30 write --no-cleanup

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados bench -p plain-pool 30 write --no-cleanup
```

## Summary

BlueStore encryption at rest in Ceph encrypts all OSD data using LUKS and AES-XTS, with keys managed by the Monitor cluster or an external KMS. In Rook, encryption is enabled by setting `encrypted: true` in the storage device set configuration and must be applied at OSD provisioning time. Monitor backups are essential when using Monitor-managed keys to ensure data recoverability.
