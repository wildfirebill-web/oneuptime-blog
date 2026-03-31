# How to Rotate CephX Keys Without Downtime

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephX, Key Rotation, Security, Zero Downtime

Description: Safely rotate CephX authentication keys for Ceph clients and daemons in Rook without causing cluster downtime or application disruptions.

---

Regular key rotation is a security best practice. In Ceph, rotating CephX keys requires care to avoid authentication failures while the new key propagates. The approach differs slightly between application client keys and daemon keys.

## Rotating Application Client Keys

The safest approach for client key rotation is a blue-green swap:

**Step 1: Generate a new key without replacing the old one**

```bash
# Export the current key
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get client.myapp -o /tmp/myapp-old.keyring

# Generate a new key value (keep the same capabilities)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth import -i - << 'EOF'
[client.myapp-new]
    key = $(ceph-authtool --gen-print-key)
    caps mon = "allow r"
    caps osd = "allow rw pool=myapp-data"
EOF
```

A simpler approach - add the new key in a separate entity:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.myapp-v2 \
  mon 'allow r' \
  osd 'allow rw pool=myapp-data'
```

**Step 2: Update the application to use the new key**

```bash
NEW_KEY=$(kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-key client.myapp-v2)

kubectl -n myapp-namespace create secret generic ceph-keyring-v2 \
  --from-literal=key="$NEW_KEY"

# Update application deployment to use new secret
kubectl -n myapp-namespace set env deploy/myapp CEPH_KEY_VERSION=v2
```

**Step 3: Delete the old key after confirming the application works**

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth del client.myapp

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth rename client.myapp-v2 client.myapp
```

## Rotating Keys In-Place

For less critical keys, rotate in-place by regenerating the key value:

```bash
# Generate a new random key
NEW_KEY=$(ceph-authtool --gen-print-key)

# Import updated keyring
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth import -i - << EOF
[client.myapp]
    key = $NEW_KEY
    caps mon = "allow r"
    caps osd = "allow rw pool=myapp-data"
EOF
```

Update the Kubernetes Secret immediately:

```bash
kubectl -n rook-ceph create secret generic ceph-client-myapp \
  --from-literal=key="$NEW_KEY" \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Rotating Daemon Keys

Ceph daemon key rotation is more complex and typically involves daemon restart. Rook handles this during upgrades. For manual rotation:

```bash
# Regenerate OSD key (requires OSD restart)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth del osd.5

kubectl -n rook-ceph rollout restart deploy/rook-ceph-osd-5
```

## Summary

Client key rotation in Ceph is safest with a blue-green approach: create a new key entity, update applications to use it, verify functionality, then delete the old key. Avoid rotating multiple keys simultaneously. Rook manages daemon key rotation automatically during upgrades, so manual daemon key rotation is rarely required.
