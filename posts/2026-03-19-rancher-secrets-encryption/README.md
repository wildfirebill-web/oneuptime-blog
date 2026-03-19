# How to Configure Secrets Encryption in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security, Encryption

Description: Learn how to configure and manage Kubernetes secrets encryption in Rancher-managed clusters using native encryption providers and external KMS integration.

Kubernetes secrets store sensitive data such as passwords, API keys, and TLS certificates. By default, these are stored as base64-encoded plaintext in etcd, which means they are not truly secure. Configuring secrets encryption ensures that this sensitive data is encrypted before it reaches etcd. This guide covers multiple approaches to secrets encryption in Rancher.

## Prerequisites

- Rancher v2.5 or later
- RKE or RKE2 managed clusters
- Admin access to Rancher
- SSH access to control plane nodes

## Step 1: Enable Built-in Secrets Encryption in RKE2

RKE2 provides the simplest way to enable secrets encryption:

```bash
# On the RKE2 server node
cat >> /etc/rancher/rke2/config.yaml << 'EOF'
secrets-encryption: true
EOF

systemctl restart rke2-server
```

Verify encryption is enabled:

```bash
rke2 secrets-encrypt status
```

Expected output:

```plaintext
Encryption Status: Enabled
Current Rotation Stage: start
```

## Step 2: Enable Secrets Encryption in RKE via Rancher

For RKE clusters managed through Rancher:

1. Go to **Cluster Management**.
2. Click the three-dot menu on the cluster.
3. Select **Edit Config** > **Edit as YAML**.
4. Add the encryption configuration:

```yaml
rancher_kubernetes_engine_config:
  services:
    kube_api:
      secrets_encryption_config:
        enabled: true
```

5. Save the changes. Rancher will update the cluster.

## Step 3: Create a Custom Encryption Configuration

For fine-grained control, create a custom encryption configuration:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: YOUR_32_BYTE_BASE64_KEY
      - identity: {}
```

Generate a secure key:

```bash
KEY=$(head -c 32 /dev/urandom | base64)
echo "Encryption key: $KEY"
```

Save the configuration on the control plane node:

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
              secret: ${KEY}
      - identity: {}
EOF

chmod 600 /etc/rancher/rke2/encryption-config.yaml
```

Configure RKE2 to use it:

```yaml
# /etc/rancher/rke2/config.yaml
kube-apiserver-arg:
  - "encryption-provider-config=/etc/rancher/rke2/encryption-config.yaml"
```

## Step 4: Encrypt All Existing Secrets

After enabling encryption, existing secrets are still stored unencrypted. Re-encrypt them:

```bash
kubectl get secrets -A -o json | kubectl replace -f -
```

For large clusters, process namespace by namespace to avoid timeouts:

```bash
for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
  echo "Re-encrypting secrets in namespace: $ns"
  kubectl get secrets -n "$ns" -o json 2>/dev/null | kubectl replace -f - 2>/dev/null
done
```

## Step 5: Verify Secrets Are Encrypted

Read a secret directly from etcd to confirm encryption:

```bash
ETCDCTL_API=3 etcdctl get /registry/secrets/default/test-secret \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  --endpoints=https://127.0.0.1:2379 | hexdump -C | head -20
```

Encrypted secrets will show binary data with the prefix `k8s:enc:aescbc:v1:key1:`.

## Step 6: Rotate Encryption Keys

Rotate keys periodically to limit the impact of a key compromise.

### For RKE2 Built-in Encryption

```bash
# Prepare rotation
rke2 secrets-encrypt prepare

# Restart the server
systemctl restart rke2-server

# Rotate the key
rke2 secrets-encrypt rotate

# Restart the server again
systemctl restart rke2-server

# Re-encrypt all secrets
rke2 secrets-encrypt reencrypt

# Check status
rke2 secrets-encrypt status
```

### For Custom Encryption Configuration

1. Add the new key before the old key:

```yaml
providers:
  - aescbc:
      keys:
        - name: key2
          secret: NEW_KEY
        - name: key1
          secret: OLD_KEY
  - identity: {}
```

2. Restart the API server.
3. Re-encrypt all secrets.
4. Remove the old key.
5. Restart the API server again.

## Step 7: Integrate with External KMS

For production environments, use an external Key Management Service instead of static keys.

### AWS KMS Plugin

Deploy the AWS KMS encryption provider plugin:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - kms:
          name: aws-kms
          endpoint: unix:///var/run/kms-plugin/socket.sock
          cachesize: 1000
          timeout: 3s
      - identity: {}
```

Deploy the KMS plugin as a static pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aws-kms-plugin
  namespace: kube-system
spec:
  containers:
  - name: kms-plugin
    image: your-registry/aws-kms-plugin:latest
    env:
    - name: AWS_KMS_KEY_ID
      value: "arn:aws:kms:us-east-1:ACCOUNT:key/KEY_ID"
    - name: AWS_REGION
      value: us-east-1
    volumeMounts:
    - name: socket
      mountPath: /var/run/kms-plugin
  volumes:
  - name: socket
    hostPath:
      path: /var/run/kms-plugin
```

### HashiCorp Vault Transit

For Vault-based key management:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - kms:
          name: vault-kms
          endpoint: unix:///var/run/vault-kms/socket.sock
          cachesize: 1000
          timeout: 3s
      - identity: {}
```

## Step 8: Audit Secrets Access

Monitor who accesses secrets with audit logging:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
  verbs: ["get", "list", "watch"]
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
  verbs: ["create", "update", "patch", "delete"]
```

This logs all secret access operations for compliance and security monitoring.

## Troubleshooting

### API Server Fails After Enabling Encryption

Check the API server logs:

```bash
journalctl -u rke2-server | grep -i encrypt
```

Common issues:
- Invalid base64 key encoding
- Incorrect file permissions on the encryption config
- Missing or incorrect file path

### Secrets Cannot Be Read After Key Change

If you removed an old key before re-encrypting, secrets encrypted with that key become unreadable. Restore the old key, restart the API server, re-encrypt all secrets, then remove the old key.

### Performance Impact

Encryption adds CPU overhead. Monitor API server latency:

```bash
kubectl top pods -n kube-system -l component=kube-apiserver
```

Use `aescbc` for the best balance of security and performance in most environments.

## Conclusion

Secrets encryption is a fundamental security measure for Kubernetes clusters. Rancher-managed RKE2 clusters make it easy to enable with a single configuration option, while custom encryption configurations and external KMS integrations provide additional flexibility for enterprise environments. Combined with regular key rotation and access auditing, secrets encryption ensures your sensitive data is protected at rest.
