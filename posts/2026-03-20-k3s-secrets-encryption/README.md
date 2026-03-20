# How to Configure K3s Secrets Encryption

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Security, Encryption, Secrets, Compliance, DevOps

Description: Learn how to enable and manage secrets encryption at rest in K3s to protect sensitive data stored in the cluster datastore.

## Introduction

By default, Kubernetes stores secrets as base64-encoded data in etcd (or SQLite in K3s) - which is **not** encryption. Anyone with access to the datastore can read all secrets. Enabling secrets encryption at rest ensures that sensitive data like passwords, API keys, and certificates is encrypted using AES-256-CBC or AES-256-GCM before being stored. This is a critical security requirement for compliance standards like PCI-DSS, HIPAA, and SOC 2.

## Understanding K3s Secrets Encryption

K3s supports two approaches:
1. **Simple `--secrets-encryption` flag**: K3s manages encryption keys automatically
2. **Custom `EncryptionConfiguration`**: Full control over encryption providers and keys

## Step 1: Enable Secrets Encryption (Simple Method)

The simplest approach - let K3s manage everything:

```bash
# Install K3s with secrets encryption enabled

curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="--secrets-encryption" \
  sh -
```

Or via config file:

```yaml
# /etc/rancher/k3s/config.yaml
secrets-encryption: true
```

Restart K3s to apply:

```bash
systemctl restart k3s
```

## Step 2: Verify Encryption is Active

```bash
# Check the generated encryption config
cat /var/lib/rancher/k3s/server/cred/encryption-config.json

# Verify a secret is encrypted in the datastore
# Create a test secret
kubectl create secret generic test-secret \
  --from-literal=password=mysecretpassword \
  -n default

# If using SQLite, check if data is encrypted
# The secret data should be unreadable (encrypted)
sqlite3 /var/lib/rancher/k3s/server/db/state.db \
  "SELECT HEX(value) FROM kine WHERE name LIKE '/registry/secrets/default/test-secret'"
# Should show hex/encrypted data, NOT plaintext YAML

# If using embedded etcd, check with etcdctl
ETCDCTL_API=3 etcdctl \
  --endpoints https://127.0.0.1:2379 \
  --cacert /var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  --cert /var/lib/rancher/k3s/server/tls/etcd/client.crt \
  --key /var/lib/rancher/k3s/server/tls/etcd/client.key \
  get /registry/secrets/default/test-secret | xxd | head -5
# Should start with "k8s:enc:aescbc:v1:" indicating encryption
```

## Step 3: Custom Encryption Configuration

For advanced control, create a custom EncryptionConfiguration:

```yaml
# /etc/rancher/k3s/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps  # Optionally encrypt ConfigMaps too
    providers:
      # AES-GCM encryption (preferred, authenticated encryption)
      - aescbc:
          keys:
            # Primary key for encryption (new secrets use this key)
            - name: key1
              # 32-byte (256-bit) AES key, base64-encoded
              secret: c2VjcmV0LWFlcy1lbmNyeXB0aW9uLWtleS0zMmJ5dGVz==
      # Fallback: identity (plaintext) for migrating existing secrets
      # Remove this after migrating all existing secrets
      - identity: {}
```

Generate a proper encryption key:

```bash
# Generate a 32-byte AES key
openssl rand -base64 32

# Or use Python
python3 -c "import os, base64; print(base64.b64encode(os.urandom(32)).decode())"
```

Update K3s to use the custom encryption config:

```yaml
# /etc/rancher/k3s/config.yaml
secrets-encryption: true
kube-apiserver-arg:
  - "encryption-provider-config=/etc/rancher/k3s/encryption-config.yaml"
```

```bash
systemctl restart k3s
```

## Step 4: Rotate Encryption Keys

K3s provides a built-in key rotation command:

```bash
# Step 1: Prepare new encryption key
# K3s generates a new key and keeps the old one for reading
k3s secrets-encrypt prepare

# Verify the prepare step created a new key
k3s secrets-encrypt status

# Step 2: Rotate - restart all K3s servers to pick up new key
systemctl restart k3s

# On HA clusters, restart each server one at a time
# Then verify all servers are using the new key
k3s secrets-encrypt status

# Step 3: Reencrypt all secrets with the new key
k3s secrets-encrypt reencrypt

# Monitor reencryption progress
kubectl get events -n kube-system | grep -i secret
k3s secrets-encrypt status

# Step 4: Remove old key (after verifying all secrets are reencrypted)
k3s secrets-encrypt rotate-keys

# Final status check
k3s secrets-encrypt status
```

## Step 5: Check Encryption Status

```bash
# K3s provides a built-in status command
k3s secrets-encrypt status

# Expected output when encryption is active and healthy:
# Encryption Status: Enabled
# Current Rotation Stage: start
#
# Server Encryption Hashes:
# SERVER         HASH
# 192.168.1.10   abc123...
```

## Step 6: Migrate Existing Secrets to Encryption

If you enabled encryption on an existing cluster, existing secrets are still in plaintext. Force re-write all secrets:

```bash
# Method 1: K3s built-in reencrypt command
k3s secrets-encrypt reencrypt

# Method 2: Manual - update all secrets to trigger re-encryption
# This touches each secret, causing it to be written back encrypted
kubectl get secrets -A -o json | \
  jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name)"' | \
  while read NS_NAME; do
    NS=$(echo $NS_NAME | cut -d/ -f1)
    NAME=$(echo $NS_NAME | cut -d/ -f2)
    kubectl -n $NS get secret $NAME -o json | \
      kubectl apply -f - --validate=false
    echo "Updated: $NS/$NAME"
  done
```

## Step 7: Verify Specific Secrets Are Encrypted

```bash
# Create a test secret
kubectl create secret generic verify-encryption-test \
  --from-literal=api-key=super-secret-api-key

# Check it's encrypted in the datastore
# For SQLite:
sqlite3 /var/lib/rancher/k3s/server/db/state.db <<'EOF'
SELECT
  name,
  HEX(SUBSTR(value, 1, 20)) as value_start
FROM kine
WHERE name LIKE '/registry/secrets/default/verify-encryption-test';
EOF

# The hex value should NOT start with "7b" (0x7b = '{', start of JSON/YAML)
# It should start with "6b3873" (k8s prefix) or AES-GCM encrypted data

# If encrypted, start will be:
# 6b38733a656e633a61657367636d3a76313a  ("k8s:enc:aesgcm:v1:")
```

## Step 8: Backup Encryption Keys

Always back up your encryption keys separately from the cluster:

```bash
#!/bin/bash
# backup-encryption-keys.sh

BACKUP_DIR="/secure-backup/k3s-encryption-$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

# Backup encryption config
cp /etc/rancher/k3s/encryption-config.yaml "$BACKUP_DIR/"

# Backup K3s auto-generated encryption config
cp /var/lib/rancher/k3s/server/cred/encryption-config.json "$BACKUP_DIR/"

# Set restrictive permissions
chmod 600 "$BACKUP_DIR/"*
chmod 700 "$BACKUP_DIR"

# Optionally encrypt the backup with GPG
gpg --symmetric --cipher-algo AES256 \
  "$BACKUP_DIR/encryption-config.yaml"

echo "Encryption keys backed up to: $BACKUP_DIR"
echo "WARNING: Losing these keys means losing access to all encrypted secrets!"
```

## Conclusion

Secrets encryption at rest is a critical security control for any K3s cluster storing sensitive data. K3s's `--secrets-encryption` flag provides a simple, managed approach, while custom EncryptionConfiguration gives full control over key material and rotation schedules. Always backup your encryption keys in a secure, separate location - losing them means permanently losing access to all encrypted secrets. For compliance requirements, combine secrets encryption with RBAC policies, audit logging, and TLS to create a comprehensive security posture.
