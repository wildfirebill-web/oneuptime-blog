# How to Enable Encryption at Rest in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security, Encryption

Description: Learn how to enable encryption at rest for Kubernetes secrets and other sensitive data in Rancher-managed clusters.

By default, Kubernetes stores secrets as base64-encoded plaintext in etcd. This means anyone with access to etcd or its backups can read all secrets in the cluster. Encryption at rest ensures that data stored in etcd is encrypted and cannot be read without the encryption key. This guide covers enabling encryption at rest in Rancher-managed clusters.

## Prerequisites

- Rancher v2.5 or later
- RKE or RKE2 managed clusters
- Admin access to Rancher
- SSH access to control plane nodes

## Step 1: Understanding Encryption at Rest

Kubernetes encryption at rest works by encrypting resources before they are written to etcd. The API server handles encryption and decryption transparently, so applications are not affected. You can encrypt specific resource types, but secrets are the most important target.

Supported encryption providers:

- **aescbc**: AES-CBC with PKCS#7 padding (recommended)
- **aesgcm**: AES-GCM (faster but key must be rotated more frequently)
- **secretbox**: XSalsa20 and Poly1305 (modern and efficient)
- **identity**: No encryption (default)

## Step 2: Enable Encryption in RKE2

RKE2 provides a simple configuration option to enable secrets encryption. On the server node, edit the RKE2 config:

```bash
cat >> /etc/rancher/rke2/config.yaml << 'EOF'
secrets-encryption: true
EOF
```

Restart RKE2:

```bash
systemctl restart rke2-server
```

This enables AES-CBC encryption for secrets automatically.

Verify encryption is active:

```bash
/var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml \
  get secrets -A -o json | head -5
```

## Step 3: Enable Encryption in RKE Clusters via Rancher

For RKE clusters provisioned through Rancher, edit the cluster YAML:

1. Go to **Cluster Management**.
2. Click the three-dot menu on the cluster.
3. Select **Edit Config** > **Edit as YAML**.
4. Add the encryption configuration:

```yaml
services:
  kube-api:
    secrets_encryption_config:
      enabled: true
```

Save the changes. Rancher will update the cluster configuration and restart the API server.

## Step 4: Create a Custom Encryption Configuration

For more control over encryption, create a custom EncryptionConfiguration:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: $(head -c 32 /dev/urandom | base64)
      - identity: {}
```

Generate the encryption key:

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
echo "Generated key: $ENCRYPTION_KEY"
```

Create the configuration file on the control plane node:

```bash
cat > /etc/rancher/rke2/encryption-config.yaml << EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

Reference it in the RKE2 configuration:

```yaml
# /etc/rancher/rke2/config.yaml

kube-apiserver-arg:
  - "encryption-provider-config=/etc/rancher/rke2/encryption-config.yaml"
```

Restart the server:

```bash
systemctl restart rke2-server
```

## Step 5: Encrypt Existing Secrets

Enabling encryption at rest only affects new secrets. To encrypt existing secrets, you need to re-write them:

```bash
kubectl get secrets -A -o json | kubectl replace -f -
```

For specific namespaces:

```bash
kubectl get secrets -n production -o json | kubectl replace -f -
kubectl get secrets -n staging -o json | kubectl replace -f -
```

## Step 6: Verify Encryption Is Active

Check that secrets are encrypted in etcd by reading directly from etcd:

```bash
# On the control plane node
ETCDCTL_API=3 etcdctl get /registry/secrets/default/my-secret \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  --endpoints=https://127.0.0.1:2379
```

If encryption is active, the output will show encrypted binary data prefixed with `k8s:enc:aescbc:v1:key1:` instead of readable plaintext.

## Step 7: Rotate Encryption Keys

Regular key rotation is a security best practice. To rotate the encryption key:

1. Generate a new key:

```bash
NEW_KEY=$(head -c 32 /dev/urandom | base64)
```

2. Update the encryption config with the new key first, keeping the old key:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key2
              secret: NEW_KEY_HERE
            - name: key1
              secret: OLD_KEY_HERE
      - identity: {}
```

3. Restart the API server:

```bash
systemctl restart rke2-server
```

4. Re-encrypt all secrets with the new key:

```bash
kubectl get secrets -A -o json | kubectl replace -f -
```

5. Remove the old key from the configuration:

```yaml
providers:
  - aescbc:
      keys:
        - name: key2
          secret: NEW_KEY_HERE
  - identity: {}
```

6. Restart the API server again.

## Step 8: Encrypt Additional Resource Types

You can encrypt other resource types beyond secrets:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: YOUR_KEY
      - identity: {}
  - resources:
      - events
    providers:
      - identity: {}
```

Be selective about what you encrypt, as encryption adds CPU overhead to every read and write operation.

## Step 9: Secure the Encryption Key

The encryption key itself must be protected:

- Restrict file permissions on the encryption config:

```bash
chmod 600 /etc/rancher/rke2/encryption-config.yaml
chown root:root /etc/rancher/rke2/encryption-config.yaml
```

- Store a backup of the key in a secure external location (Vault, KMS).
- Use Kubernetes KMS provider for key management in cloud environments.
- Never commit encryption keys to version control.

## Troubleshooting

### API Server Fails to Start After Enabling Encryption

Check the API server logs:

```bash
journalctl -u rke2-server | grep -i encrypt
```

Common issues include invalid base64 encoding in the key or incorrect file paths.

### Cannot Read Secrets After Key Rotation

If the old key was removed before re-encrypting all secrets, some secrets may be unreadable. Restore the old key to the configuration, restart the API server, re-encrypt, and then remove the old key.

## Conclusion

Encryption at rest protects sensitive data stored in etcd from unauthorized access. Whether you use RKE2's built-in secrets encryption or a custom encryption configuration, enabling this feature is essential for production clusters. Combine encryption at rest with regular key rotation and secure key storage for comprehensive data protection.
