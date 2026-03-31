# How to Configure State Store Encryption at Rest with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Encryption, State Store, Security, Key Management

Description: Learn how to configure Dapr state store encryption at rest using component-level encryption settings, key rotation, and integration with cloud KMS providers.

---

## State Store Encryption Approaches

Dapr supports two complementary encryption strategies for state stores: client-side encryption (Dapr encrypts values before storing them) and server-side encryption (the backing store encrypts data on disk). For maximum security, enable both. This guide focuses on Dapr's built-in client-side encryption configuration, which ensures data is encrypted even if the backing store is compromised.

## Enabling Client-Side Encryption

Dapr's client-side state encryption is configured in the component definition using `primaryEncryptionKey` and optionally `secondaryEncryptionKey` for key rotation:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: encrypted-statestore
  namespace: production
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis-master:6379"
  - name: redisPassword
    secretKeyRef:
      name: redis-secret
      key: password
  - name: primaryEncryptionKey
    secretKeyRef:
      name: state-encryption-keys
      key: primary-key
  - name: secondaryEncryptionKey
    secretKeyRef:
      name: state-encryption-keys
      key: secondary-key
```

Dapr uses AES-256-GCM for encryption. The key must be a 32-byte base64-encoded string.

## Generating Encryption Keys

```bash
# Generate a 256-bit key
PRIMARY_KEY=$(openssl rand -base64 32)
SECONDARY_KEY=$(openssl rand -base64 32)

# Store in Kubernetes secret
kubectl create secret generic state-encryption-keys \
  --namespace production \
  --from-literal=primary-key="$PRIMARY_KEY" \
  --from-literal=secondary-key="$SECONDARY_KEY"
```

## Integrating with AWS KMS for Key Management

For production environments, store encryption keys in a KMS and rotate them automatically:

```bash
# Create a KMS key
aws kms create-key \
  --description "Dapr state store encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --region us-east-1

# Generate a data key and store it
aws kms generate-data-key \
  --key-id alias/dapr-state-key \
  --key-spec AES_256 \
  --query 'Plaintext' \
  --output text | \
kubectl create secret generic state-encryption-keys \
  --namespace production \
  --from-literal=primary-key=-
```

## Using Dapr Secrets API for Key Retrieval

Reference keys from a secrets store rather than a Kubernetes secret:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: encrypted-statestore
spec:
  type: state.postgresql
  version: v2
  metadata:
  - name: connectionString
    secretKeyRef:
      name: pg-secret
      key: connectionString
  - name: primaryEncryptionKey
    secretKeyRef:
      name: vault-kv-dapr
      key: state-primary-key
```

Where `vault-kv-dapr` is a Dapr secret component backed by HashiCorp Vault:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: vault-kv-dapr
spec:
  type: secretstores.hashicorp.vault
  version: v1
  metadata:
  - name: vaultAddr
    value: "https://vault.internal:8200"
  - name: vaultToken
    secretKeyRef:
      name: vault-token
      key: token
```

## Key Rotation Procedure

Rotate the primary key without losing access to existing state:

```bash
# Step 1: Move current primary to secondary
kubectl patch secret state-encryption-keys -n production \
  --type=json \
  -p='[{"op": "replace", "path": "/data/secondary-key", "value": "'$(kubectl get secret state-encryption-keys -n production -o jsonpath='{.data.primary-key}')'"}]'

# Step 2: Generate and set new primary key
NEW_KEY=$(openssl rand -base64 32 | base64)
kubectl patch secret state-encryption-keys -n production \
  --type=json \
  -p='[{"op": "replace", "path": "/data/primary-key", "value": "'$NEW_KEY'"}]'

# Step 3: Restart pods to pick up new keys
kubectl rollout restart deployment -n production -l dapr.io/enabled=true
```

Dapr decrypts data with the secondary key if the primary key fails, enabling zero-downtime rotation.

## Verifying Encryption

Confirm data is encrypted in the backing store by inspecting values directly:

```bash
# Connect to Redis and check a key - value should be unreadable binary
kubectl exec -n redis redis-master-0 -- redis-cli GET "dapr||myapp||mykey"
# Output: binary/encrypted data, not JSON
```

## Summary

Dapr state store encryption is enabled by adding `primaryEncryptionKey` and `secondaryEncryptionKey` to component metadata, with Dapr performing AES-256-GCM encryption before values leave the sidecar. Key rotation uses the two-key scheme: promote the current primary to secondary, then set a new primary key, and restart pods without downtime. Integrating with cloud KMS or HashiCorp Vault centralizes key management and enables automatic rotation policies.
