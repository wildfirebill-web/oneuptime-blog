# How to Enable RKE2 Secret Encryption

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Secret Encryption, Security, Encryption at Rest, Rancher

Description: Learn how to enable and manage secret encryption at rest in RKE2 to protect sensitive cluster data stored in etcd.

Kubernetes secrets are base64-encoded by default, which provides no actual encryption - anyone with etcd access can decode them. Enabling encryption at rest ensures that secrets stored in etcd are encrypted with a strong cipher. RKE2 provides a built-in secret encryption mechanism that can be enabled and rotated. This guide covers the complete lifecycle of secret encryption in RKE2.

## Prerequisites

- RKE2 server cluster running
- Cluster admin access
- Understanding of AES encryption concepts

## Understanding RKE2 Secret Encryption

RKE2 uses the Kubernetes API server's encryption provider framework. When enabled:

1. New secrets are encrypted before being stored in etcd
2. Existing secrets remain unencrypted until re-encrypted
3. Encryption uses AES-CBC with PKCS#7 padding (or AES-GCM for newer versions)
4. Encryption keys can be rotated

## Step 1: Enable Secret Encryption

RKE2 provides a built-in command to manage secret encryption:

```bash
# Enable secret encryption on the cluster

sudo rke2 secrets-encrypt enable

# This command:
# 1. Generates an encryption key
# 2. Creates the encryption configuration file
# 3. Updates the API server to use the encryption configuration
# 4. Restarts the API server

# Check the status
sudo rke2 secrets-encrypt status
```

## Step 2: Verify Encryption Status

```bash
# Check if encryption is enabled
sudo rke2 secrets-encrypt status

# Expected output when enabled:
# Encryption Status: Enabled
# Current Rotation Stage: start

# Check the encryption configuration file
sudo cat /var/lib/rancher/rke2/server/cred/encryption-state.json

# View the API server encryption config
sudo cat /var/lib/rancher/rke2/server/tls/encryption-config.yaml
```

## Step 3: Re-encrypt Existing Secrets

After enabling encryption, existing secrets are still unencrypted. Re-encrypt them:

```bash
# Prepare for re-encryption (updates API server config)
sudo rke2 secrets-encrypt prepare

# Wait for all API server instances to reload
# For HA clusters, this requires all server nodes to be updated

# Rotate the re-encryption (actually re-encrypts existing secrets)
sudo rke2 secrets-encrypt rotate

# Complete the rotation
sudo rke2 secrets-encrypt reencrypt

# Check the final status
sudo rke2 secrets-encrypt status
```

## Step 4: Manual Encryption Configuration

For more control, you can manually configure encryption:

```bash
# Generate a strong AES-256 encryption key
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
echo "New encryption key: $ENCRYPTION_KEY"

# Create the encryption config
cat <<EOF | sudo tee /etc/rancher/rke2/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    # AES-GCM is more secure than AES-CBC (available in K8s 1.21+)
    - aesgcm:
        keys:
        - name: key1
          secret: ${ENCRYPTION_KEY}
    # Keep identity as fallback for reading existing unencrypted secrets
    - identity: {}
  # Also encrypt ConfigMaps with sensitive data
  - resources:
    - configmaps
    providers:
    - aesgcm:
        keys:
        - name: key1
          secret: ${ENCRYPTION_KEY}
    - identity: {}
EOF

# Reference the config in RKE2 configuration
cat >> /etc/rancher/rke2/config.yaml << EOF
kube-apiserver-arg:
  - "encryption-provider-config=/etc/rancher/rke2/encryption-config.yaml"
EOF

sudo systemctl restart rke2-server
```

## Step 5: Re-encrypt Existing Secrets After Manual Configuration

```bash
# After enabling encryption, re-encrypt all existing secrets
# This forces all secrets to be written with the new encryption

kubectl get secrets -A -o json | kubectl replace -f -

# Verify a specific secret is encrypted
# The raw etcd value should be encrypted, not base64-readable
# Check using etcdctl
ETCD_CERT_DIR="/var/lib/rancher/rke2/server/tls/etcd"
/var/lib/rancher/rke2/bin/etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=${ETCD_CERT_DIR}/server-ca.crt \
  --cert=${ETCD_CERT_DIR}/client.crt \
  --key=${ETCD_CERT_DIR}/client.key \
  get /registry/secrets/default/my-secret | \
  strings | head -5

# The value should be unreadable binary if encrypted correctly
# If you see readable text, encryption is NOT working
```

## Step 6: Rotate Encryption Keys

Periodically rotate encryption keys for security compliance:

```bash
# Step 1: Generate a new encryption key
NEW_KEY=$(head -c 32 /dev/urandom | base64)

# Step 2: Add the new key first, keep the old key for decryption
cat <<EOF | sudo tee /etc/rancher/rke2/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    # New key is first (used for encryption)
    - aesgcm:
        keys:
        - name: key2
          secret: ${NEW_KEY}
        # Keep old key for decrypting existing secrets
        - name: key1
          secret: ${OLD_KEY}
    - identity: {}
EOF

# Step 3: Restart API server to use new config
sudo systemctl restart rke2-server

# Step 4: Re-encrypt all secrets with the new key
kubectl get secrets -A -o json | kubectl replace -f -

# Step 5: Remove the old key once all secrets are re-encrypted
cat <<EOF | sudo tee /etc/rancher/rke2/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    # Only the new key remains
    - aesgcm:
        keys:
        - name: key2
          secret: ${NEW_KEY}
    - identity: {}
EOF

sudo systemctl restart rke2-server
echo "Key rotation complete"
```

## Step 7: Verify Encryption is Working

```bash
# Create a test secret
kubectl create secret generic test-encryption \
  --from-literal=password=supersecret \
  -n default

# Read the encrypted value from etcd
ETCD_CERT_DIR="/var/lib/rancher/rke2/server/tls/etcd"
/var/lib/rancher/rke2/bin/etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=${ETCD_CERT_DIR}/server-ca.crt \
  --cert=${ETCD_CERT_DIR}/client.crt \
  --key=${ETCD_CERT_DIR}/client.key \
  get /registry/secrets/default/test-encryption | hexdump -C | head -5

# The output should show binary encrypted data, not readable text

# Verify Kubernetes can still read the secret
kubectl get secret test-encryption -o jsonpath='{.data.password}' | base64 -d
# Should output: supersecret

# Clean up
kubectl delete secret test-encryption
```

## Conclusion

Enabling secret encryption at rest in RKE2 is an essential security control that protects sensitive credentials and configuration data from unauthorized access to the etcd datastore. The built-in `rke2 secrets-encrypt` command simplifies the process of enabling, verifying, and rotating encryption keys. For compliance frameworks like HIPAA, PCI DSS, and government security standards, encryption at rest is a mandatory requirement. Once enabled, ensure you have a secure backup of your encryption keys - losing them means losing access to all encrypted secrets in your cluster.
