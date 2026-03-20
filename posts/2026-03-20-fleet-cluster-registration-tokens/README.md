# How to Configure Fleet Cluster Registration Tokens

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fleet, GitOps, Rancher, Kubernetes, Registration

Description: Learn how to create and manage Fleet ClusterRegistrationToken resources to securely onboard new clusters into your Fleet management plane.

## Introduction

Before Fleet can manage a cluster, that cluster must be registered with the Fleet manager. ClusterRegistrationTokens are the mechanism through which clusters authenticate and join the Fleet management plane. Each token generates a registration manifest that is applied to the downstream cluster, causing the Fleet agent to install and establish a connection.

This guide covers creating registration tokens, managing their lifecycles, and following security best practices.

## Prerequisites

- Fleet installed in Rancher
- Admin access to Fleet manager
- `kubectl` access to Fleet manager namespace
- Target clusters where you want to install Fleet agents

## Understanding ClusterRegistrationTokens

A ClusterRegistrationToken:
1. Is created in a specific Fleet namespace (workspace)
2. Has a configurable time-to-live (TTL)
3. Generates a URL or manifest for agent installation
4. Can be used by multiple clusters

When a cluster uses a token to register, Fleet creates a `Cluster` resource in the corresponding namespace.

## Creating a ClusterRegistrationToken

### Basic Token with Default TTL

```yaml
# registration-token-default.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterRegistrationToken
metadata:
  name: default-token
  namespace: fleet-default
spec:
  # Token expires after 24 hours (default)
  ttl: 24h
```

```bash
# Create the token
kubectl apply -f registration-token-default.yaml

# Check the token status and get the secret name
kubectl get clusterregistrationtoken default-token -n fleet-default -o yaml
```

### Non-Expiring Token for Edge/Bulk Registration

```yaml
# registration-token-permanent.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterRegistrationToken
metadata:
  name: edge-registration
  namespace: fleet-default
  labels:
    purpose: edge-cluster-registration
    managed-by: fleet-admin
spec:
  # Set TTL to 0 for a non-expiring token
  # Use with caution - rotate periodically
  ttl: 0s
```

### Short-Lived Token for One-Time Registration

```yaml
# registration-token-short.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterRegistrationToken
metadata:
  name: onetime-registration
  namespace: fleet-default
spec:
  # 1 hour - use immediately and discard
  ttl: 1h
```

## Retrieving the Registration Manifest

After creating a token, retrieve the agent installation manifest:

```bash
# Get the namespace where the manifest is stored
kubectl get clusterregistrationtoken default-token \
  -n fleet-default \
  -o jsonpath='{.status.manifestNamespace}'

# Get the secret containing the registration values
MANIFEST_NS=$(kubectl get clusterregistrationtoken default-token \
  -n fleet-default \
  -o jsonpath='{.status.manifestNamespace}')

# The registration manifest URL for the downstream cluster
# Access it via the Fleet manager API endpoint
echo "Registration namespace: ${MANIFEST_NS}"
```

### Getting the Helm Values for Agent Installation

```bash
# Get Fleet agent installation values from the token
kubectl get secret \
  $(kubectl get clusterregistrationtoken default-token -n fleet-default \
    -o jsonpath='{.status.secretName}') \
  -n fleet-default \
  -o jsonpath='{.data.values}' | base64 -d
```

## Installing Fleet Agent Using the Token

### Using Helm on the Downstream Cluster

```bash
# Step 1: Switch kubectl context to the downstream cluster
kubectl config use-context my-downstream-cluster

# Step 2: Get the Fleet manager URL and token
FLEET_MANAGER_URL="https://rancher.example.com"
CLUSTER_NAMESPACE="fleet-default"

# Step 3: Install Fleet agent
helm repo add fleet https://rancher.github.io/fleet-helm-charts/
helm repo update

helm install fleet-agent fleet/fleet-agent \
  --namespace cattle-fleet-system \
  --create-namespace \
  --set apiServerURL="${FLEET_MANAGER_URL}" \
  --set token="$(kubectl get clusterregistrationtoken default-token \
    -n fleet-default \
    --context=fleet-manager-context \
    -o jsonpath='{.status.secretName}')" \
  --set clusterNamespace="${CLUSTER_NAMESPACE}" \
  --set clusterName="my-downstream-cluster"
```

### Using kubectl apply with Registration Manifest

```bash
# In Rancher, get the import command from the cluster registration UI
# The command looks like:
kubectl apply -f https://rancher.example.com/v3/import/token-xxxx.yaml
```

## Managing Token Lifecycle

### Listing All Registration Tokens

```bash
# List all tokens across all namespaces
kubectl get clusterregistrationtokens -A

# Get token expiration info
kubectl get clusterregistrationtokens -A \
  -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: expires={.spec.ttl}{"\n"}{end}'
```

### Rotating Tokens

```bash
# Delete the old token
kubectl delete clusterregistrationtoken old-token -n fleet-default

# Create a new token with a new name
cat <<EOF | kubectl apply -f -
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterRegistrationToken
metadata:
  name: edge-registration-v2
  namespace: fleet-default
spec:
  ttl: 0s
EOF

# Update edge clusters to use the new token
# (Existing registered clusters don't need to re-register)
```

## Creating Workspace-Specific Tokens

Use separate tokens per workspace for isolation:

```bash
# Create tokens for each workspace
for workspace in fleet-team-alpha fleet-team-beta fleet-production; do
  cat <<EOF | kubectl apply -f -
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterRegistrationToken
metadata:
  name: ${workspace}-token
  namespace: ${workspace}
spec:
  ttl: 0s
EOF
  echo "Created token for workspace: ${workspace}"
done
```

## Verifying Cluster Registration

After a cluster uses a token to register:

```bash
# Check that the cluster appears in Fleet
kubectl get clusters.fleet.cattle.io -n fleet-default

# Verify the cluster is connected
kubectl get cluster my-downstream-cluster \
  -n fleet-default \
  -o jsonpath='{.status.agent}'
```

## Security Best Practices

1. **Use short-lived tokens** for one-time cluster registrations
2. **Use non-expiring tokens** only for automated/edge deployments with proper monitoring
3. **Rotate tokens regularly** even for edge deployments
4. **Store tokens in a secrets manager** (HashiCorp Vault, AWS Secrets Manager)
5. **Monitor token usage** by watching for new Cluster resource creation
6. **Use workspace-specific tokens** to prevent cross-workspace registrations

```bash
# Monitor for new cluster registrations
kubectl get clusters.fleet.cattle.io -n fleet-default -w
```

## Conclusion

ClusterRegistrationTokens are the secure gateway for onboarding Kubernetes clusters into Fleet management. By choosing appropriate TTLs for your use case — short-lived for one-time registrations and longer-lived for automated edge deployments — you balance security with operational convenience. Regular token rotation and workspace-level token isolation ensure that your Fleet management plane remains secure even as you scale to hundreds or thousands of managed clusters.
