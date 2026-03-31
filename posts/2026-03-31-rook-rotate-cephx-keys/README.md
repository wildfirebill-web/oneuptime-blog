# How to Rotate CephX Keys in Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Security, CephX, Key Rotation, Kubernetes

Description: Learn how to rotate CephX authentication keys in Rook-Ceph for security compliance - covering client key rotation, admin key update, and Kubernetes secret synchronization.

---

## Why Rotate CephX Keys

Key rotation is a security best practice that limits the blast radius of credential compromise. In regulated environments, quarterly or annual key rotation may be required by policy.

## Understanding CephX Keys in Rook

Rook stores CephX keys as Kubernetes Secrets:

```bash
kubectl -n rook-ceph get secrets | grep ceph
# rook-ceph-admin-keyring
# rook-ceph-mon
# rook-ceph-osd-<id>
# rook-ceph-csi-rbd-node
# rook-ceph-csi-rbd-provisioner
```

## Step 1: Rotate a Client Key

For application client keys (e.g., CSI provisioner):

```bash
# Generate a new key
NEW_KEY=$(kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph-authtool --gen-print-key)

# Update the key in Ceph's auth database
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth caps client.csi-rbd-provisioner \
  mon 'profile rbd' \
  osd 'profile rbd'

# Get the current key and note it for rollback
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get client.csi-rbd-provisioner
```

## Step 2: Generate and Apply New CSI Keys

Rook provides a mechanism to regenerate CSI keys by deleting and re-creating the relevant secrets:

```bash
# Delete the existing CSI secrets - Rook will regenerate them
kubectl -n rook-ceph delete secret rook-ceph-csi-rbd-node
kubectl -n rook-ceph delete secret rook-ceph-csi-rbd-provisioner

# The Rook operator will create new secrets with new keys
kubectl -n rook-ceph get secrets | grep csi
```

## Step 3: Rotate the Admin Keyring

```bash
# Create a new admin key
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.admin \
  mon 'allow *' osd 'allow *' mgr 'allow *' mds 'allow *' \
  -o /tmp/new-admin.keyring

# Export the new key
NEW_ADMIN_KEY=$(kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-key client.admin)

# Update the Kubernetes secret
kubectl -n rook-ceph patch secret rook-ceph-admin-keyring \
  -p "{\"data\":{\"keyring\":\"$(echo "[client.admin]\n\tkey = $NEW_ADMIN_KEY" | base64 -w0)\"}}"
```

## Step 4: Update Application Secrets

If applications store the CephX key (e.g., in a PersistentVolume provisioner), update their secrets:

```bash
NEW_KEY_B64=$(kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-key client.my-app | base64 -w0)

kubectl -n app-namespace patch secret ceph-app-secret \
  -p "{\"data\":{\"key\":\"$NEW_KEY_B64\"}}"

kubectl -n app-namespace rollout restart deployment/my-app
```

## Step 5: Verify Authentication Works

After rotation, verify that all consumers can still authenticate:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
kubectl -n rook-ceph get pods -l app=rook-ceph-osd
# Ensure all OSDs remain up and running

# Test PVC provisioning still works
kubectl apply -f test-pvc.yaml
kubectl get pvc test-pvc
```

## Summary

CephX key rotation in Rook involves updating keys in Ceph's auth database, refreshing Kubernetes Secrets, and restarting any components that hold the old credentials. Rook simplifies CSI key rotation by regenerating secrets automatically when the old ones are deleted.
