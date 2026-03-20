# How to Enable Longhorn Volume Encryption at Rest - Volume

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Encryption, Kubernetes, Storage, Security, LUKS, SUSE Rancher

Description: Learn how to enable Longhorn volume encryption at rest using LUKS to protect sensitive data stored in Longhorn persistent volumes on Kubernetes clusters.

---

Longhorn supports volume encryption at rest using LUKS (Linux Unified Key Setup). Encryption protects data stored on disk from physical access threats and meets compliance requirements for sensitive workloads.

---

## How Longhorn Encryption Works

Longhorn uses the Linux kernel's `dm-crypt` module with LUKS to encrypt volume data at the block device level. Each encrypted volume gets a unique encryption key stored in a Kubernetes Secret.

---

## Step 1: Create an Encryption Key Secret

```yaml
# longhorn-crypto-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: longhorn-crypto
  namespace: longhorn-system
stringData:
  # Base64-encoded encryption key (at least 256 bits)
  CRYPTO_KEY_VALUE: "this-is-a-very-long-random-secret-key-replace-this"
  # Encryption algorithm (aes-xts-plain64 is FIPS-approved)
  CRYPTO_KEY_CIPHER: "aes-xts-plain64"
  # Key size in bits
  CRYPTO_KEY_SIZE: "256"
  # Hash function for PBKDF2
  CRYPTO_PBKDF: "argon2i"
```

```bash
kubectl apply -f longhorn-crypto-secret.yaml
```

---

## Step 2: Create an Encrypted StorageClass

```yaml
# storageclass-encrypted.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-encrypted
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  fromBackup: ""
  # Enable encryption
  encrypted: "true"
  # Reference the secret containing the crypto key
  csi.storage.k8s.io/provisioner-secret-name: longhorn-crypto
  csi.storage.k8s.io/provisioner-secret-namespace: longhorn-system
  csi.storage.k8s.io/node-publish-secret-name: longhorn-crypto
  csi.storage.k8s.io/node-publish-secret-namespace: longhorn-system
  csi.storage.k8s.io/node-stage-secret-name: longhorn-crypto
  csi.storage.k8s.io/node-stage-secret-namespace: longhorn-system
```

```bash
kubectl apply -f storageclass-encrypted.yaml
```

---

## Step 3: Create an Encrypted PVC

```yaml
# encrypted-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: encrypted-data
  namespace: my-app
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-encrypted
  resources:
    requests:
      storage: 10Gi
```

---

## Step 4: Verify Encryption

After the PVC is bound and mounted, confirm encryption is active:

```bash
# Check the Longhorn volume status in the UI or via CLI
kubectl get volume -n longhorn-system | grep encrypted

# On the node where the volume is mounted, verify LUKS is active
lsblk | grep dm-crypt

# Check the Longhorn volume details
kubectl get lhvolume <volume-name> -n longhorn-system -o yaml | grep encrypted
```

---

## Rotating Encryption Keys

To rotate the encryption key, create a new Secret and trigger a key rotation:

```bash
# Update the secret with the new key
kubectl patch secret longhorn-crypto \
  -n longhorn-system \
  --type merge \
  -p '{"stringData":{"CRYPTO_KEY_VALUE":"new-super-secret-key"}}'
```

---

## Best Practices

- Store encryption keys in an external secrets manager (HashiCorp Vault, AWS Secrets Manager) and use External Secrets Operator to sync them to Kubernetes.
- Use different encryption keys per environment (dev, staging, production).
- Test encrypted volume restore from backup before relying on it for production data.
- Encryption has a small performance overhead (typically 5-10%) - benchmark your workload before production deployment.
