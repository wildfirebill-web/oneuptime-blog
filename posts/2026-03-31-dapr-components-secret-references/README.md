# How to Secure Dapr Components with Secret References

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Security, Secret, Component, Configuration

Description: Learn how to use Dapr secret references to keep sensitive credentials out of component YAML files by resolving values from secret stores at runtime.

---

## Overview

Dapr component YAML files often contain sensitive values such as database passwords, API keys, and connection strings. Hardcoding these in plain text is a security risk. Dapr supports `secretKeyRef` fields in component metadata so you can reference secrets stored in Kubernetes Secrets, Vault, or any other supported secret store instead of embedding them directly.

## Bootstrap Secret Store

The Kubernetes secret store is always available in Kubernetes mode with no additional configuration. It is the recommended bootstrap store for resolving secrets used by other components.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kubernetes-secret-store
  namespace: default
spec:
  type: secretstores.kubernetes
  version: v1
```

## Referencing Secrets in Components

Use `secretKeyRef` in any metadata field that contains a sensitive value:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: my-postgres
  namespace: default
spec:
  type: state.postgresql
  version: v1
  metadata:
  - name: connectionString
    secretKeyRef:
      name: postgres-secret
      key: connectionString
auth:
  secretStore: kubernetes
```

Create the Kubernetes secret:

```bash
kubectl create secret generic postgres-secret \
  --from-literal=connectionString="host=mydb user=app password=securepass dbname=orders sslmode=require"
```

## Using Vault as the Secret Store

For more advanced secret management, use HashiCorp Vault:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: vault-store
  namespace: default
spec:
  type: secretstores.hashicorp.vault
  version: v1
  metadata:
  - name: vaultAddr
    value: "https://vault.example.com:8200"
  - name: vaultToken
    secretKeyRef:
      name: vault-token
      key: token
```

Then reference Vault secrets in your component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: redis-state
  namespace: default
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis:6379"
  - name: redisPassword
    secretKeyRef:
      name: redis-credentials
      key: password
auth:
  secretStore: vault-store
```

## RBAC for Secret Access

Restrict which Kubernetes service accounts can read secrets used by Dapr components:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dapr-secret-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["postgres-secret", "redis-credentials"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dapr-secret-reader-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: default
roleRef:
  kind: Role
  name: dapr-secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## Verifying Secret Resolution

Check that the component loaded successfully and that secrets resolved correctly:

```bash
kubectl get component my-postgres -o yaml
dapr components --kubernetes --namespace default
```

If the secret fails to resolve, the sidecar logs a clear error on startup.

## Summary

Dapr secret references eliminate the need to embed credentials in component YAML files. By pointing metadata fields at named secrets in Kubernetes or an external vault, you keep sensitive values out of source control and apply fine-grained RBAC to control which workloads can access which secrets.
