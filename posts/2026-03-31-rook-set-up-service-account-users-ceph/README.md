# How to Set Up Service Account Users in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Authentication

Description: Learn how to create and manage Ceph service account users for automated workloads, CI/CD pipelines, and Kubernetes applications in Rook clusters.

---

## What Are Service Account Users in Ceph

A service account user in Ceph is a `client.*` authentication entity created specifically for an automated system, application, or service - not for a human operator. Service account users follow least-privilege principles, have programmatically generated names, and are typically mapped to a Kubernetes ServiceAccount for RBAC-controlled access to the credentials.

## Naming Conventions

Use descriptive names that identify the owning service:

```text
client.backup-agent
client.prometheus-exporter
client.app-webapp-prod
client.cicd-pipeline
```

Avoid generic names like `client.myapp` that become unclear over time.

## Creating a Service Account User

Create a service account user with appropriate capabilities:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

```bash
# Backup service account - read only
ceph auth get-or-create client.backup-agent \
  mon 'allow r' \
  osd 'allow r'

# Application service account - pool scoped
ceph auth get-or-create client.webapp-prod \
  mon 'allow r' \
  osd 'allow rw pool=webapp-prod-data'

# Monitoring service account
ceph auth get-or-create client.prometheus \
  mon 'allow r' \
  mgr 'allow r'
```

## Storing Keys as Kubernetes Secrets

After creating the user, store the key as a Kubernetes Secret with a clear naming convention:

```bash
KEY=$(ceph auth print-key client.webapp-prod)

kubectl create secret generic ceph-svc-webapp-prod \
  --from-literal=key="${KEY}" \
  --from-literal=userID="client.webapp-prod" \
  -n webapp-prod-namespace
```

## Binding Secrets to Kubernetes Service Accounts

Use RBAC to ensure only the correct Kubernetes ServiceAccount can read the Ceph secret:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ceph-key-reader
  namespace: webapp-prod-namespace
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["ceph-svc-webapp-prod"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ceph-key-reader-binding
  namespace: webapp-prod-namespace
subjects:
- kind: ServiceAccount
  name: webapp-prod-sa
  namespace: webapp-prod-namespace
roleRef:
  kind: Role
  name: ceph-key-reader
  apiGroup: rbac.authorization.k8s.io
```

## Labeling Secrets for Auditing

Add labels to Ceph secret objects for easier auditing:

```bash
kubectl label secret ceph-svc-webapp-prod \
  ceph-user="client.webapp-prod" \
  team="webapp" \
  environment="prod" \
  -n webapp-prod-namespace
```

## Service Account Lifecycle Management

When a service is decommissioned, clean up both the Ceph user and the Kubernetes Secret:

```bash
# Remove Kubernetes Secret
kubectl delete secret ceph-svc-webapp-prod -n webapp-prod-namespace

# Remove Ceph user
ceph auth del client.webapp-prod
```

## Auditing Service Account Users

List all Ceph service account users and cross-reference with active Kubernetes Secrets:

```bash
# List all client users
ceph auth ls --format json | jq -r '.auth_dump[].entity' | grep "^client\." | sort

# List all Ceph-related Kubernetes Secrets
kubectl get secrets --all-namespaces -l ceph-user --no-headers \
  -o custom-columns="NS:.metadata.namespace,NAME:.metadata.name,USER:.metadata.labels.ceph-user"
```

## Summary

Ceph service account users are `client.*` entities created for automated workloads rather than human operators. Use descriptive naming, apply least-privilege capabilities, store keys as Kubernetes Secrets with RBAC bindings, and label secrets for auditability. Maintain a lifecycle process to delete both the Ceph user and Kubernetes Secret when a service is decommissioned.
