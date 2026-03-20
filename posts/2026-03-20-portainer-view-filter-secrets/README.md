# How to View and Filter Secrets in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Secrets, Security, Management, DevOps

Description: Learn how to view, search, and manage Kubernetes Secrets in Portainer while maintaining security hygiene and avoiding accidental secret exposure.

## Introduction

Managing Kubernetes Secrets requires balancing operational visibility with security. You need to find the right secret quickly, verify it has the right keys, and understand which workloads depend on it — all without accidentally exposing sensitive values. Portainer provides a Secrets management interface with appropriate value masking, while kubectl offers CLI tools for scripting and automation. This guide covers viewing and filtering Secrets safely in Portainer.

## Prerequisites

- Portainer with Kubernetes environment
- Secrets deployed in one or more namespaces
- Appropriate RBAC permissions to view secrets

## Step 1: Navigate to Secrets in Portainer

1. Select your Kubernetes environment in Portainer
2. Select a namespace from the namespace dropdown
3. Click **ConfigMaps & Secrets** in the sidebar
4. Select the **Secrets** tab

The list displays:
```
Name                   Namespace    Type                        Keys    Created
database-credentials   production   Opaque                      5       2 days ago
registry-credentials   production   kubernetes.io/dockerconfigjson  1   5 days ago
tls-production         production   kubernetes.io/tls           2       30 days ago
app-api-keys           production   Opaque                      8       1 week ago
```

## Step 2: Search and Filter Secrets in Portainer

In the Secrets list:

1. Use the **search bar** to filter by name:
   - Type `tls` to find TLS-related secrets
   - Type `registry` to find registry credential secrets
   - Type `database` to find database credential secrets

2. Use the **namespace filter** to narrow results:
   - Select `production` to see only production secrets
   - Select `All namespaces` for cluster-wide view

3. Filter by **Type** if available:
   - `Opaque` — general purpose secrets
   - `kubernetes.io/tls` — TLS certificates
   - `kubernetes.io/dockerconfigjson` — registry credentials

## Step 3: View Secret Details (Keys Only, Not Values)

Click on a Secret to see its details. Portainer shows:
- Secret name, namespace, type
- Creation timestamp and labels
- **Keys** listed without revealing values
- Values are masked by default (shown as `****` or hidden)

Some Portainer versions allow revealing values via a toggle — use this with caution in shared environments or screen sharing sessions.

## Step 4: View Secret Metadata via kubectl

```bash
# List all secrets in a namespace
kubectl get secrets -n production

# View secret keys without values
kubectl describe secret database-credentials -n production

# Example output:
# Name:         database-credentials
# Namespace:    production
# Labels:       app=my-app
# Annotations:  <none>
# Type:         Opaque
# Data
# ====
# DATABASE_HOST:      35 bytes
# DATABASE_NAME:      14 bytes
# DATABASE_PASSWORD:  24 bytes
# DATABASE_PORT:      4 bytes
# DATABASE_USER:      8 bytes

# List only secret names
kubectl get secrets -n production -o name

# Filter by label
kubectl get secrets -n production -l app=my-app

# Filter by type
kubectl get secrets -n production \
  --field-selector type=kubernetes.io/tls
```

## Step 5: Decode Secret Values (Use with Caution)

Only decode secret values when absolutely necessary, in a secure environment:

```bash
# Decode a specific key value
kubectl get secret database-credentials -n production \
  -o jsonpath='{.data.DATABASE_PASSWORD}' | base64 --decode
echo   # Add newline

# Decode all values in a secret (sensitive output!)
kubectl get secret database-credentials -n production \
  -o json | jq -r '.data | to_entries[] |
  "\(.key): \(.value | @base64d)"'

# Get secret as JSON (values are base64-encoded)
kubectl get secret database-credentials -n production -o json | \
  jq '.data | keys'    # Just the keys, no values

# Check if a specific key exists
kubectl get secret database-credentials -n production \
  -o jsonpath='{.data.DATABASE_PASSWORD}' | \
  xargs -I{} sh -c '[ -n "{}" ] && echo "Key exists" || echo "Key missing"'
```

## Step 6: Find Secrets Referenced by Workloads

Identify which deployments use specific secrets:

```bash
# Find deployments referencing a secret via env vars
kubectl get deployments -n production -o json | \
  jq -r '.items[] | select(
    .spec.template.spec.containers[].env[]? |
    select(.valueFrom.secretKeyRef.name == "database-credentials")
  ) | .metadata.name'

# Find deployments using envFrom with a secret
kubectl get deployments -n production -o json | \
  jq -r '.items[] | select(
    .spec.template.spec.containers[].envFrom[]? |
    select(.secretRef.name == "database-credentials")
  ) | .metadata.name'

# Find pods with a secret mounted as volume
kubectl get pods -n production -o json | \
  jq -r '.items[] | select(
    .spec.volumes[]? | select(.secret.secretName == "database-credentials")
  ) | .metadata.name'
```

## Step 7: Audit Secret Access

Track who accessed secrets and when:

```bash
# Check audit logs for secret access (requires audit logging configured)
# View events related to secrets in a namespace
kubectl get events -n production | grep Secret

# List secrets with their last-modified time
kubectl get secrets -n production \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}'

# Find recently created secrets
kubectl get secrets -n production \
  --sort-by='.metadata.creationTimestamp' | tail -5
```

## Step 8: Identify Expiring TLS Certificates

Find TLS secrets that are expiring soon:

```bash
# Check TLS certificate expiry for all TLS secrets
for secret in $(kubectl get secrets -n production \
  --field-selector type=kubernetes.io/tls -o name); do
  name=$(echo $secret | sed 's/secret\///')
  echo -n "$name: "
  kubectl get secret $name -n production \
    -o jsonpath='{.data.tls\.crt}' | \
    base64 --decode | \
    openssl x509 -noout -enddate 2>/dev/null || echo "Unable to parse"
done
```

## Step 9: Clean Up Unused Secrets

Find secrets that no workload references:

```bash
#!/bin/bash
NAMESPACE=production

echo "All secrets:"
kubectl get secrets -n $NAMESPACE -o name | \
  grep -v "default-token\|service-account-token" | \
  sed 's/secret\///' | sort > /tmp/all-secrets.txt

echo "Referenced secrets:"
kubectl get pods,deployments,statefulsets -n $NAMESPACE -o json | \
  jq -r '[
    .items[].spec | (
      .template.spec // . |
      (
        (.containers[].env[]?.valueFrom.secretKeyRef.name // empty),
        (.containers[].envFrom[]?.secretRef.name // empty),
        (.volumes[]?.secret.secretName // empty)
      )
    )
  ] | unique | .[]' | sort > /tmp/referenced-secrets.txt

echo "Potentially unused secrets:"
comm -23 /tmp/all-secrets.txt /tmp/referenced-secrets.txt
```

## Conclusion

Viewing and filtering Secrets in Portainer provides operational visibility without accidentally exposing sensitive values. The default masked view keeps values hidden during normal operations; reveal values only when necessary and never in screen shares or logs. Use kubectl's `describe` command (which shows key names and byte counts without values) for safe auditing. Regularly audit secret usage to identify unused secrets for cleanup and check TLS certificate expiry to prevent unexpected outages.
