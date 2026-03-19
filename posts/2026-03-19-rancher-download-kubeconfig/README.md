# How to Download kubeconfig from Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, kubeconfig, kubectl, CLI

Description: Step-by-step guide to downloading and managing kubeconfig files from Rancher for accessing your Kubernetes clusters.

A kubeconfig file is your credential and connection configuration for accessing a Kubernetes cluster. Rancher generates kubeconfig files that include authentication tokens and cluster endpoints. This guide covers every method for downloading kubeconfig files from Rancher and managing them effectively.

## Method 1: Download from the Rancher UI

### Step 1: Navigate to the Cluster

Log into your Rancher instance and click on the cluster you want to access from the main dashboard.

### Step 2: Open the Cluster Dashboard

Once you are in the cluster view, look for the kubeconfig button. In Rancher v2.7+, click the **Copy KubeConfig to Clipboard** button at the top of the cluster explorer page, or find it under the kebab menu (three dots).

### Step 3: Save the Configuration

Paste the copied content into a file:

```bash
# Create the .kube directory if it doesn't exist
mkdir -p ~/.kube

# Paste the kubeconfig content (or save from the UI download)
# Save as the default config or a named file
vim ~/.kube/my-cluster.yaml
```

### Step 4: Set the KUBECONFIG Environment Variable

```bash
export KUBECONFIG=~/.kube/my-cluster.yaml
kubectl get nodes
```

## Method 2: Download via the Rancher API

This is the best method for automation and scripting.

```bash
export RANCHER_URL="https://rancher.example.com"
export RANCHER_TOKEN="token-xxxxx:yyyyyyyyyyyyyyyy"
export CLUSTER_ID="c-m-abc12345"

curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/clusters/${CLUSTER_ID}?action=generateKubeconfig" | jq -r '.config' > ~/.kube/my-cluster.yaml

echo "Kubeconfig saved to ~/.kube/my-cluster.yaml"
```

Verify it works:

```bash
export KUBECONFIG=~/.kube/my-cluster.yaml
kubectl cluster-info
```

## Method 3: Download via the Rancher CLI

```bash
rancher login https://rancher.example.com --token ${RANCHER_TOKEN}

# List clusters to find the name
rancher clusters ls

# Download kubeconfig for a specific cluster
rancher clusters kubeconfig production > ~/.kube/production.yaml
```

## Understanding the kubeconfig Structure

A Rancher-generated kubeconfig looks like this:

```yaml
apiVersion: v1
kind: Config
clusters:
- name: my-cluster
  cluster:
    server: https://rancher.example.com/k8s/clusters/c-m-abc12345
    certificate-authority-data: LS0tLS1...
contexts:
- name: my-cluster
  context:
    cluster: my-cluster
    user: my-cluster
current-context: my-cluster
users:
- name: my-cluster
  user:
    token: kubeconfig-user-xxxx:yyyyyyyy
```

Key points:

- The **server** URL routes through the Rancher proxy by default
- The **token** is a Rancher-issued kubeconfig token (different from your API key)
- The **certificate-authority-data** contains the Rancher server's CA certificate

## Downloading kubeconfig with Direct Endpoint

If you have the Authorized Cluster Endpoint (ACE) enabled, the kubeconfig may include a direct connection to the cluster API server:

```yaml
clusters:
- name: my-cluster
  cluster:
    server: https://10.0.1.100:6443
    certificate-authority-data: LS0tLS1...
- name: my-cluster-rancher
  cluster:
    server: https://rancher.example.com/k8s/clusters/c-m-abc12345
```

You can choose which endpoint to use by switching contexts:

```bash
kubectl config use-context my-cluster        # Direct access
kubectl config use-context my-cluster-rancher # Through Rancher proxy
```

## Downloading kubeconfig for All Clusters

Automate downloading kubeconfig for every cluster:

```bash
#!/bin/bash

RANCHER_URL="https://rancher.example.com"
RANCHER_TOKEN="token-xxxxx:yyyyyyyyyyyyyyyy"

mkdir -p ~/.kube/rancher

# Get all cluster IDs and names
clusters=$(curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/clusters" | jq -r '.data[] | "\(.id)|\(.name)"')

while IFS='|' read -r id name; do
  echo "Downloading kubeconfig for ${name} (${id})..."
  curl -s -k -X POST \
    -H "Authorization: Bearer ${RANCHER_TOKEN}" \
    "${RANCHER_URL}/v3/clusters/${id}?action=generateKubeconfig" | \
    jq -r '.config' > ~/.kube/rancher/${name}.yaml
done <<< "$clusters"

echo "All kubeconfigs saved to ~/.kube/rancher/"
ls -la ~/.kube/rancher/
```

## Merging Multiple kubeconfig Files

### Temporary Merge with KUBECONFIG Variable

```bash
export KUBECONFIG=~/.kube/rancher/production.yaml:~/.kube/rancher/staging.yaml:~/.kube/rancher/dev.yaml

# Now kubectl can see all clusters
kubectl config get-contexts
```

### Permanent Merge into a Single File

```bash
# Back up existing config
cp ~/.kube/config ~/.kube/config.backup

# Merge all rancher kubeconfigs
export KUBECONFIG=$(find ~/.kube/rancher -name "*.yaml" | tr '\n' ':')
kubectl config view --flatten > ~/.kube/config.merged

# Replace the default config
mv ~/.kube/config.merged ~/.kube/config

# Verify
kubectl config get-contexts
```

### Rename Merged Contexts

After merging, rename contexts for clarity:

```bash
kubectl config rename-context c-m-abc12345 production
kubectl config rename-context c-m-def67890 staging
kubectl config rename-context c-m-ghi11111 development

kubectl config get-contexts
```

## Automating kubeconfig Refresh

Rancher kubeconfig tokens expire after a configurable period. Set up automatic refresh:

```bash
#!/bin/bash
# refresh-kubeconfigs.sh

RANCHER_URL="https://rancher.example.com"
RANCHER_TOKEN="token-xxxxx:yyyyyyyyyyyyyyyy"

clusters=$(curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/clusters" | jq -r '.data[] | "\(.id)|\(.name)"')

while IFS='|' read -r id name; do
  curl -s -k -X POST \
    -H "Authorization: Bearer ${RANCHER_TOKEN}" \
    "${RANCHER_URL}/v3/clusters/${id}?action=generateKubeconfig" | \
    jq -r '.config' > ~/.kube/rancher/${name}.yaml
done <<< "$clusters"

# Rebuild merged config
export KUBECONFIG=$(find ~/.kube/rancher -name "*.yaml" | tr '\n' ':')
kubectl config view --flatten > ~/.kube/config

echo "$(date): Kubeconfigs refreshed" >> /var/log/kubeconfig-refresh.log
```

Add a cron job to run daily:

```bash
0 6 * * * /opt/scripts/refresh-kubeconfigs.sh
```

## Security Considerations

### Token Expiration

Rancher kubeconfig tokens have a default TTL set in the Rancher settings. Check your setting:

```bash
curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/settings/kubeconfig-default-token-TTL-minutes" | jq '.value'
```

### File Permissions

Always restrict kubeconfig file permissions:

```bash
chmod 600 ~/.kube/config
chmod 600 ~/.kube/rancher/*.yaml
```

### Avoid Committing kubeconfig to Git

Add kubeconfig patterns to your `.gitignore`:

```
**/kubeconfig*
**/*.kubeconfig
.kube/
```

### Revoking kubeconfig Tokens

If a kubeconfig is compromised, revoke the associated token:

```bash
# Find kubeconfig tokens
curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/tokens?kind=kubeconfig" | jq '.data[] | {id, description, expired}'

# Delete a specific token
curl -s -k -X DELETE \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/tokens/kubeconfig-user-xxxx"
```

## Troubleshooting

### "Unable to connect to the server"

Verify the server URL in your kubeconfig:

```bash
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
```

Test connectivity:

```bash
curl -sk "$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')/healthz"
```

### "Unauthorized" After Token Expiry

Regenerate the kubeconfig using any of the methods described above.

### "x509: certificate signed by unknown authority"

Either add the CA certificate to your kubeconfig or skip verification:

```bash
kubectl config set-cluster my-cluster --insecure-skip-tls-verify=true
```

## Summary

Downloading kubeconfig from Rancher can be done through the UI, the API, or the CLI. For production workflows, use the API method to automate kubeconfig generation and refresh. Merge multiple kubeconfigs into a single file for easy context switching, set proper file permissions, and configure automatic token refresh to avoid authentication failures.
