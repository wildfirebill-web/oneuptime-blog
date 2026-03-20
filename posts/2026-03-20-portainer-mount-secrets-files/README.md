# How to Mount Secrets as Files in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Secrets, Volumes, Security, DevOps

Description: Learn how to mount Kubernetes Secrets as files inside containers using Portainer for secure credential injection, TLS certificates, and SSH key management.

## Introduction

Mounting Secrets as files in containers is more secure than environment variable injection for highly sensitive data. Files are stored in `tmpfs` (in-memory) on the node - they never touch disk. Applications read credentials directly from the filesystem, which is a well-established pattern for credentials files, TLS certificates, and SSH keys. Portainer supports Secret volume mounts through its application form and YAML editor.

## Prerequisites

- Portainer with Kubernetes environment
- An existing Secret with the data to mount
- A deployment to configure

## Why Files Over Environment Variables

Secret files stored in tmpfs offer security advantages:

```text
In-memory (tmpfs)     - Files never written to node disk
Auto-updated          - Volume mounts sync when Secret changes (after ~60s)
Process isolation     - Files not visible via /proc/<pid>/environ
Access control        - File permissions (chmod) restrict access within container
Structured data       - Better for certificates, keys, credential files
```

Environment variables are visible via process inspection; files are more restricted.

## Step 1: Create the Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
  namespace: production
type: Opaque
stringData:
  database.conf: |
    [database]
    host=postgres.production.svc.cluster.local
    port=5432
    user=app_user
    password=super-secret-db-password
    dbname=myapp

  api-keys.json: |
    {
      "stripe": "sk_live_abc123",
      "sendgrid": "SG.abc123",
      "twilio": {
        "account_sid": "AC123",
        "auth_token": "abc123def456"
      }
    }
```

## Step 2: Mount Secret via Portainer Form

When creating or editing an application:

1. Go to the **Volumes** section
2. Click **+ Add volume**
3. Configure:
   ```text
   Type: Secret
   Secret name: app-credentials
   Mount path: /run/secrets
   Read-only: Yes
   ```
4. Deploy the application

Files appear at the mount path:
- `/run/secrets/database.conf`
- `/run/secrets/api-keys.json`

## Step 3: Mount Secret via YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: my-app:latest
          volumeMounts:
            - name: app-credentials
              mountPath: /run/secrets
              readOnly: true
      volumes:
        - name: app-credentials
          secret:
            secretName: app-credentials
            defaultMode: 0400   # Owner read-only
```

## Step 4: Mount TLS Certificate Files

For HTTPS clients that need to verify TLS certificates:

```yaml
# TLS Secret

apiVersion: v1
kind: Secret
metadata:
  name: app-tls
  namespace: production
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    MIIFazCCA1O...
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    MIIEvQIBAD...
    -----END PRIVATE KEY-----
```

Mount in an application:

```yaml
volumeMounts:
  - name: tls-certs
    mountPath: /etc/ssl/certs/app
    readOnly: true

volumes:
  - name: tls-certs
    secret:
      secretName: app-tls
      items:
        - key: tls.crt
          path: tls.crt
          mode: 0444
        - key: tls.key
          path: tls.key
          mode: 0400   # Private key - read-only by owner only
```

## Step 5: Mount SSH Private Key

For applications that authenticate to Git or other SSH services:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-ssh-key
  namespace: production
type: Opaque
stringData:
  id_rsa: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    b3BlbnNzaC1rZXktdjEAAAAA...
    -----END OPENSSH PRIVATE KEY-----
  known_hosts: "github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GkZD"
```

Mount with proper permissions:

```yaml
volumeMounts:
  - name: ssh-keys
    mountPath: /home/app/.ssh
    readOnly: true

volumes:
  - name: ssh-keys
    secret:
      secretName: git-ssh-key
      defaultMode: 0400   # SSH requires strict permissions
```

## Step 6: Mount Single Secret Key as Named File

Use `subPath` to mount a single key without affecting other files in a directory:

```yaml
volumeMounts:
  - name: db-password
    mountPath: /run/secrets/db_password   # Single file path
    subPath: password                      # Key in the Secret
    readOnly: true

volumes:
  - name: db-password
    secret:
      secretName: database-credentials
```

Result: The file `/run/secrets/db_password` contains the value of the `password` key.

## Step 7: Read Secret Files in Your Application

```python
# Python: Reading credentials from mounted secret files
import json

# Read database config
with open('/run/secrets/database.conf', 'r') as f:
    import configparser
    config = configparser.ConfigParser()
    config.read_string(f.read())
    db_password = config['database']['password']

# Read API keys JSON
with open('/run/secrets/api-keys.json', 'r') as f:
    api_keys = json.load(f)
    stripe_key = api_keys['stripe']
```

```javascript
// Node.js: Reading credentials from mounted secret files
const fs = require('fs');

// Read a single secret value
const dbPassword = fs.readFileSync('/run/secrets/db_password', 'utf8').trim();

// Read JSON credentials
const apiKeys = JSON.parse(fs.readFileSync('/run/secrets/api-keys.json', 'utf8'));
const stripeKey = apiKeys.stripe;
```

## Step 8: Auto-Update Behavior

Secret volume mounts auto-update when the Secret changes:

```bash
# Update secret value
kubectl edit secret app-credentials -n production
# or
kubectl apply -f updated-secret.yaml

# Files in the pod update automatically (within ~60 seconds)
# kubelet sync period controls the delay

# Verify the file updated
kubectl exec <pod-name> -n production -- \
  stat /run/secrets/database.conf    # Check modification time

# Application must re-read the file to use new values
# Some apps support config file watching (e.g., Vault Agent)
# Others require restart:
kubectl rollout restart deployment/my-app -n production
```

## Conclusion

Mounting Secrets as files is the most secure way to inject sensitive credentials into Kubernetes pods. Files reside in `tmpfs` (never on disk), support strict file permissions, and auto-update when Secrets change. Use this pattern for TLS certificates, SSH keys, credential files, and any data that structured file formats handle better than individual environment variables. Combine with proper `defaultMode` settings and `readOnly: true` mounts to enforce least-privilege access within containers.
