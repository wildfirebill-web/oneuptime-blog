# How to Download Kubeconfig from Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, kubectl, kubeconfig, DevOps

Description: Learn how to download a kubeconfig file from Portainer for kubectl CLI access, including how to configure it for multiple environments and set up context switching.

## Introduction

Portainer generates user-specific kubeconfig files that provide scoped kubectl access to Kubernetes clusters managed by Portainer. This is especially useful in Portainer Business Edition where namespace-level access control is enforced. This guide walks through the full process of downloading and configuring your kubeconfig.

## Prerequisites

- Portainer BE with Kubernetes environment(s) configured
- kubectl access enabled by an admin for your user account
- kubectl installed on your local machine (v1.20+)

## Step 1: Enable kubectl Access (Admin Task)

An administrator must first enable kubectl access before users can download kubeconfig files:

1. Log into Portainer as admin.
2. Select the Kubernetes environment.
3. Go to environment **Settings** (gear icon).
4. Enable **Allow users to use kubectl shell**.
5. Set a kubeconfig expiry time (optional but recommended).
6. Save.

## Step 2: Download Your Kubeconfig via the UI

As a regular user:

1. Log into Portainer.
2. Click your **username** in the top-right corner of the page.
3. Click **My account**.
4. Scroll down to the **Kubeconfig** section.
5. From the **Environment** dropdown, select the cluster you need access to.
6. Click **Download kubeconfig**.

The file will be downloaded as `portainer-kubeconfig.yaml`.

## Step 3: Download Kubeconfig via the Portainer API

You can also download kubeconfig programmatically:

```bash
# Step 1: Authenticate and get JWT token

TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"myuser","password":"mypassword"}' | jq -r '.jwt')

# Step 2: Download kubeconfig for endpoint ID 1
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/config" \
  -o portainer-kubeconfig.yaml

echo "Kubeconfig downloaded."
```

## Step 4: Install and Use the Kubeconfig

```bash
# Option A: Replace default kubeconfig
cp portainer-kubeconfig.yaml ~/.kube/config

# Option B: Use as an additional context alongside existing configs
export KUBECONFIG=~/.kube/config:~/portainer-kubeconfig.yaml

# Option C: Merge into existing kubeconfig
KUBECONFIG=~/.kube/config:~/portainer-kubeconfig.yaml \
  kubectl config view --flatten > /tmp/merged.yaml
mv /tmp/merged.yaml ~/.kube/config

# Make it permanent in your shell profile
echo 'export KUBECONFIG=~/.kube/config:~/portainer-kubeconfig.yaml' >> ~/.zshrc
source ~/.zshrc
```

## Step 5: Verify the Kubeconfig Contents

```bash
# View the kubeconfig structure
cat portainer-kubeconfig.yaml

# Check available contexts
kubectl config get-contexts

# Example output:
# CURRENT   NAME                        CLUSTER             AUTHINFO            NAMESPACE
# *         portainer-production        portainer-cluster   portainer-user      production
#           portainer-staging           portainer-cluster   portainer-user-2    staging
```

## Step 6: Switch Between Contexts

```bash
# Switch to a specific Portainer context
kubectl config use-context portainer-production

# Confirm current context
kubectl config current-context

# Test access
kubectl get namespaces
kubectl get pods -n production

# Use a specific context for a single command (without switching)
kubectl get pods -n staging --context=portainer-staging
```

## Handling Kubeconfig Expiry

When the kubeconfig token expires, you will see this error:

```text
error: You must be logged in to the server (Unauthorized)
```

To resolve:

```bash
# Re-download fresh kubeconfig from Portainer API
TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"myuser","password":"mypassword"}' | jq -r '.jwt')

curl -s -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/config" \
  -o portainer-kubeconfig.yaml

# Re-merge with your kubeconfig
KUBECONFIG=~/.kube/config:~/portainer-kubeconfig.yaml \
  kubectl config view --flatten > /tmp/merged.yaml && mv /tmp/merged.yaml ~/.kube/config

echo "Kubeconfig refreshed."
```

## Automating Kubeconfig Refresh

```bash
#!/bin/bash
# refresh-kubeconfig.sh - Run as a cron job to keep kubeconfig fresh

PORTAINER_URL="https://portainer.example.com"
PORTAINER_USER="myuser"
PORTAINER_PASS="mypassword"
ENDPOINT_ID=1

TOKEN=$(curl -s -X POST "${PORTAINER_URL}/api/auth" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"${PORTAINER_USER}\",\"password\":\"${PORTAINER_PASS}\"}" | jq -r '.jwt')

curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/kubernetes/config" \
  -o ~/.kube/portainer-kubeconfig.yaml

echo "Kubeconfig refreshed at $(date)"
```

```bash
# Add to crontab to refresh every 6 hours
# crontab -e
0 */6 * * * /home/user/refresh-kubeconfig.sh >> /var/log/kubeconfig-refresh.log 2>&1
```

## Conclusion

Downloading kubeconfig from Portainer gives developers convenient CLI access to Kubernetes clusters with appropriate access controls enforced by Portainer's RBAC system. Use the UI for one-time downloads and the API approach for automated, scheduled refreshes. Always set a kubeconfig expiry in Portainer to maintain a strong security posture.
