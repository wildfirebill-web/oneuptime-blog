# How to Create Secrets via YAML Manifest in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Secrets, YAML, Security

Description: Learn how to create Kubernetes Secrets by applying YAML manifests in Portainer, with proper base64 encoding.

## When to Use YAML for Secrets

Use the YAML editor when:
- Migrating Secrets from another cluster.
- Creating complex secret types (TLS, dockerconfigjson).
- Scripting Secret creation and applying via Portainer's YAML editor.
- You need to set multiple keys in a structured format.

**Important**: In raw YAML, values in the `data` field must be base64-encoded. Use `stringData` to avoid manual encoding.

## Using `stringData` (No Encoding Required)

The `stringData` field accepts plain text values and Kubernetes encodes them automatically:

```yaml
# Using stringData - NO base64 encoding needed

apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:
  # Plain text values - Kubernetes encodes them for you
  DB_PASSWORD: "s3cur3_p@ssword!"
  API_KEY: "myapikey-12345-abcde"
  JWT_SECRET: "my-very-long-jwt-secret-key-here"
```

## Using `data` Field (Manual Base64 Encoding)

```bash
# Encode values before putting them in the data field
echo -n "mypassword" | base64
# Output: bXlwYXNzd29yZA==
```

```yaml
# Using data - values MUST be base64-encoded
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  DB_PASSWORD: bXlwYXNzd29yZA==
  API_KEY: bXlhcGlrZXktMTIzNDUtYWJjZGU=
```

## TLS Secret

```yaml
# TLS Secret for HTTPS certificates
apiVersion: v1
kind: Secret
metadata:
  name: my-tls-secret
  namespace: production
type: kubernetes.io/tls
data:
  # Both fields are required and must be base64-encoded
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>
```

```bash
# Create TLS secret from actual cert files (easier via CLI)
kubectl create secret tls my-tls-secret \
  --cert=fullchain.pem \
  --key=privkey.pem \
  --namespace=production \
  --dry-run=client -o yaml > tls-secret.yaml
# Then paste the output into Portainer's YAML editor
```

## Docker Registry Secret (for imagePullSecrets)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

```bash
# Generate the base64 docker config
kubectl create secret docker-registry registry-credentials \
  --docker-server=registry.mycompany.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --dry-run=client -o yaml
# Copy the output YAML into Portainer
```

## Applying YAML in Portainer

1. Go to **ConfigMaps & Secrets > Add Secret**.
2. Switch to **YAML editor** mode.
3. Paste your YAML.
4. Click **Create**.

Alternatively, use the Portainer kubectl shell for direct application.

## Conclusion

YAML-based Secret creation gives you precise control and is ideal for complex secret types. Always prefer `stringData` over `data` when writing YAML manually to avoid base64 encoding errors.
