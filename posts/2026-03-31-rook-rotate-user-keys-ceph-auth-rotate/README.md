# How to Rotate User Keys with ceph auth rotate

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Authentication

Description: Learn how to rotate Ceph user authentication keys using ceph auth rotate to meet security requirements without losing user capabilities.

---

## Why Rotate User Keys

Key rotation is a security best practice that limits the exposure window if a key is compromised. Rotating Ceph user keys invalidates the old key and generates a new one while preserving all existing capabilities. This is useful for:

- Periodic security rotations mandated by policy
- Suspected key compromise
- Off-boarding a team member who had access to keys
- Satisfying compliance requirements

## The ceph auth rotate Command

`ceph auth rotate` generates a new random key for an existing user and returns the updated keyring. The user's capabilities remain unchanged.

Access from the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

Rotate a key:

```bash
ceph auth rotate client.myapp
```

Output:

```text
[client.myapp]
    key = AQZ...==
```

The new key is different from the previous one. Any application using the old key will immediately fail authentication.

## Saving the New Keyring

Export the new keyring to a file immediately after rotation:

```bash
ceph auth rotate client.myapp
ceph auth get client.myapp -o /tmp/myapp-new.keyring
```

## Updating Kubernetes Secrets After Rotation

In Rook environments, after rotating a key you must update the corresponding Kubernetes Secrets. Otherwise, pods using the old key will start failing.

Step-by-step rotation workflow:

```bash
# Step 1: Rotate the key in Ceph
ceph auth rotate client.myapp

# Step 2: Get the new key
NEW_KEY=$(ceph auth print-key client.myapp)

# Step 3: Update the Kubernetes Secret
kubectl -n myapp-namespace patch secret ceph-myapp-key \
  --type='json' \
  -p="[{\"op\": \"replace\", \"path\": \"/data/key\", \"value\": \"$(echo -n $NEW_KEY | base64)\"}]"

# Step 4: Restart affected pods to pick up the new secret
kubectl -n myapp-namespace rollout restart deployment myapp
```

## Zero-Downtime Key Rotation Pattern

For production workloads, a zero-downtime rotation requires a brief window where both old and new keys are valid. Ceph does not natively support dual-key validity, so you must coordinate the update carefully:

```bash
# Step 1: Create a temporary parallel user for migration
ceph auth get-or-create client.myapp-new \
  mon 'allow r' \
  osd 'allow rw pool=appdata'

# Step 2: Update application config to use client.myapp-new
# (deploy new pods with new key)

# Step 3: After all pods are on the new user, remove the old one
ceph auth del client.myapp

# Step 4: Rename or recreate as client.myapp if needed
ceph auth get-or-create client.myapp \
  mon 'allow r' \
  osd 'allow rw pool=appdata'
ceph auth del client.myapp-new
```

## Alternative: Generate and Import a Custom Key

If you need a specific key value (for example to match an HSM-generated key):

```bash
# Generate a new key using ceph-authtool
ceph-authtool --gen-print-key

# Apply the custom key (requires setting it directly)
# Note: ceph auth rotate always generates a random key
# For custom keys, use ceph-authtool to create a keyring then import
```

## Verifying the Rotation

After rotation, confirm the key changed:

```bash
# Check the key in Ceph
ceph auth print-key client.myapp

# Check the key in the Kubernetes Secret
kubectl -n myapp-namespace get secret ceph-myapp-key \
  -o jsonpath='{.data.key}' | base64 -d
```

Both should match after the update.

## Scheduling Periodic Rotation

Automate key rotation using a Kubernetes CronJob:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rotate-ceph-key
  namespace: rook-ceph
spec:
  schedule: "0 3 1 * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: rotate
            image: rook/ceph:v1.15.0
            command: ["/bin/bash", "-c", "ceph auth rotate client.myapp"]
```

## Summary

`ceph auth rotate` generates a new random key for a Ceph user while preserving all capabilities. After rotation, immediately update Kubernetes Secrets and restart affected pods to prevent authentication failures. For zero-downtime rotation, use a parallel user migration pattern to avoid service interruption.
