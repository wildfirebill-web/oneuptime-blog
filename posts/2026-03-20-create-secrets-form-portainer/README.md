# How to Create Secrets via Form in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Secret, Security, Configuration Management

Description: Learn how to create Kubernetes Secrets using Portainer's form interface for secure storage of sensitive configuration data.

## What Are Kubernetes Secrets?

Secrets are similar to ConfigMaps but intended for sensitive data like passwords, API keys, tokens, and certificates. They are base64-encoded in etcd and can be restricted with RBAC. Portainer handles the encoding automatically when you use the form interface.

## Creating a Secret via Form in Portainer

1. Select your Kubernetes environment.
2. Go to **ConfigMaps & Secrets** or **Secrets**.
3. Click **Add Secret**.
4. Fill in:
   - **Name**: A descriptive name (e.g., `db-credentials`, `api-keys`).
   - **Namespace**: Select the target namespace.
   - **Type**: Choose the secret type (see below).
5. Add key-value entries.
6. Click **Create Secret**.

**Important**: Portainer automatically base64-encodes the values you enter in the form. You do **not** need to encode them yourself.

## Secret Types

| Type | Use Case |
|------|----------|
| `Opaque` | Generic key-value secrets |
| `kubernetes.io/tls` | TLS certificates |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/ssh-auth` | SSH private keys |

## Example: Creating an Opaque Secret for a Database

In the form, add these key-value pairs:

| Key | Value |
|-----|-------|
| `POSTGRES_USER` | `myapp_user` |
| `POSTGRES_PASSWORD` | `s3cur3p@ssw0rd!` |
| `POSTGRES_DB` | `myapp_production` |

Portainer creates:

```yaml
# What Portainer generates (values are automatically base64-encoded)

apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  POSTGRES_USER: bXlhcHBfdXNlcg==
  POSTGRES_PASSWORD: czNjdXIzcEBzc3cwcmQh
  POSTGRES_DB: bXlhcHBfcHJvZHVjdGlvbg==
```

## Creating a Secret via CLI

```bash
# Create from literal values (kubectl handles base64 encoding)
kubectl create secret generic db-credentials \
  --from-literal=POSTGRES_USER=myapp_user \
  --from-literal=POSTGRES_PASSWORD="s3cur3p@ssw0rd!" \
  --from-literal=POSTGRES_DB=myapp_production \
  --namespace=production

# Create from a file (useful for certificates, SSH keys)
kubectl create secret generic ssl-cert \
  --from-file=tls.crt=server.crt \
  --from-file=tls.key=server.key \
  --namespace=production
```

## Using the Secret in a Deployment

After creating the secret in Portainer, attach it to your deployment:

```yaml
spec:
  containers:
    - name: app
      image: myapp:latest
      env:
        # Inject specific secret keys as environment variables
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: POSTGRES_PASSWORD
      envFrom:
        # Inject all keys from the secret
        - secretRef:
            name: db-credentials
```

## Verifying (Without Exposing Values)

```bash
# Check the secret exists without showing values
kubectl get secret db-credentials --namespace=production

# Check which keys are in the secret
kubectl get secret db-credentials --namespace=production \
  -o jsonpath='{.data}' | jq 'keys'
```

## Conclusion

Portainer's Secret form editor handles base64 encoding automatically, reducing the chance of encoding errors. Always use Secrets (not ConfigMaps) for passwords, API keys, and certificates, and restrict access to the Secrets namespace with RBAC.
