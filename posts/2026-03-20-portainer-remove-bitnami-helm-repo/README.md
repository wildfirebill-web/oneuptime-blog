# How to Remove the Default Bitnami Helm Repository in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Helm, Security, DevOps

Description: Learn how to remove the default Bitnami Helm repository from Portainer to simplify your chart catalog or enforce organizational policies on approved repositories.

## Introduction

Portainer ships with the Bitnami Helm repository pre-configured. While Bitnami provides high-quality charts, some organizations prefer to curate their own approved repository list, reduce the chart catalog to only internal sources, or remove external dependencies for air-gapped deployments. This guide shows how to remove the default Bitnami repository from Portainer.

## Prerequisites

- Portainer CE or BE with a Kubernetes environment
- Admin access to Portainer
- Understanding that removing the repo removes it from the Helm charts UI (existing releases are unaffected)

## Why Remove the Bitnami Repository?

Common reasons include:

- **Security hardening**: Limit chart sources to internally vetted repositories
- **Air-gapped environments**: No external internet access for chart index fetching
- **Organizational policy**: Only allow charts from approved private repositories
- **Reduce noise**: Simplify the chart catalog for developers to only show relevant charts
- **Performance**: Fewer repos to index means faster Helm chart page loading

## Step 1: Navigate to Helm Repository Settings

1. Log into Portainer as an administrator.
2. Select your **Kubernetes** environment.
3. Click the **gear icon** to open environment settings.
4. Scroll down to the **Helm repository** section.

You will see the default Bitnami repository listed:
- Name: `bitnami`
- URL: `https://charts.bitnami.com/bitnami`

## Step 2: Remove the Bitnami Repository

1. Click the **trash can** icon next to the Bitnami repository entry.
2. Confirm the removal when prompted.
3. Click **Save environment settings**.

After removal, Bitnami charts will no longer appear in the Helm charts list for this environment.

> **Note**: Removing the repository does **not** affect existing Helm releases deployed from Bitnami charts. Running applications will continue to work normally. It only prevents new deployments from that repository going forward.

## Step 3: Remove via the Portainer API

For scripted or automated removal across multiple environments:

```bash
# Authenticate

TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' | jq -r '.jwt')

# List current Helm repositories for endpoint 1
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/helm/repositories" | jq .

# Find the Bitnami repo ID from the response
REPO_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/helm/repositories" | \
  jq -r '.[] | select(.url | contains("bitnami")) | .id')

echo "Bitnami repo ID: $REPO_ID"

# Delete the Bitnami repository
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/helm/repositories/${REPO_ID}"

echo "Bitnami repository removed."
```

## Step 4: Bulk Remove from Multiple Environments

If you manage multiple Kubernetes environments:

```bash
#!/bin/bash
# remove-bitnami-from-all-envs.sh

PORTAINER_URL="https://portainer.example.com"
TOKEN="your-jwt-token"

# Get all endpoint IDs
ENDPOINTS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints" | jq -r '.[] | select(.Type == 7) | .Id')

for ENDPOINT_ID in $ENDPOINTS; do
  echo "Processing endpoint $ENDPOINT_ID..."

  # Find Bitnami repo ID
  REPO_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
    "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/kubernetes/helm/repositories" | \
    jq -r '.[] | select(.url | contains("bitnami")) | .id // empty')

  if [ -n "$REPO_ID" ]; then
    # Delete it
    curl -s -X DELETE -H "Authorization: Bearer $TOKEN" \
      "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/kubernetes/helm/repositories/${REPO_ID}"
    echo "  Removed Bitnami repo (ID: $REPO_ID) from endpoint $ENDPOINT_ID"
  else
    echo "  No Bitnami repo found on endpoint $ENDPOINT_ID"
  fi
done

echo "Done."
```

## Step 5: Replace with Approved Internal Repositories

After removing Bitnami, add only your approved repositories:

```bash
# Add your internal Helm repository
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/helm/repositories" \
  -d '{
    "url": "https://charts.internal.company.com",
    "name": "company-approved"
  }'
```

## Verifying the Removal

```bash
# Confirm Bitnami no longer appears in the repository list
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/helm/repositories" | \
  jq '.[] | .url'
# Should not include https://charts.bitnami.com/bitnami
```

## Conclusion

Removing the default Bitnami Helm repository from Portainer is a simple but important step when enforcing an approved-repositories-only policy. Existing deployments are unaffected by this change. After removal, add only your vetted and approved chart repositories to maintain control over which applications can be deployed in your Kubernetes environments.
