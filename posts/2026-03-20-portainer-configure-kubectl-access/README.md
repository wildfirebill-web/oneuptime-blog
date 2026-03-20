# How to Configure kubectl Access in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, kubectl, DevOps, Security

Description: Learn how to configure kubectl access through Portainer so developers can use the Kubernetes CLI with proper RBAC permissions tied to their Portainer credentials.

## Introduction

Portainer can act as a kubectl proxy, issuing Kubeconfig files scoped to individual users' access rights. This allows teams to use the standard `kubectl` CLI while Portainer enforces namespace-level RBAC policies. This guide covers enabling kubectl access, configuring per-user scopes, and distributing kubeconfig files to your team.

## Prerequisites

- Portainer BE (Business Edition) with a Kubernetes environment
- Admin access to Portainer
- kubectl installed on developer machines
- Users already created in Portainer

## Step 1: Enable kubectl Access in Portainer

1. Log into Portainer as admin.
2. Select your **Kubernetes** environment.
3. Click the **gear icon** (environment settings) next to the environment name.
4. Scroll to the **Security** section.
5. Enable **Allow users to use kubectl** by toggling it on.
6. Optionally set a **kubeconfig expiry** (e.g., 24h for short-lived credentials).
7. Click **Save environment settings**.

## Step 2: Configure User Access Scopes

Each Portainer user can have their kubectl access scoped to specific namespaces:

1. Go to **Namespaces** within the Kubernetes environment.
2. Click on a namespace.
3. Assign it to a **team** or specific **users**.
4. Users will only see and interact with namespaces they have been granted access to.

## Step 3: Users Download Their Kubeconfig

Users with kubectl access enabled can download their personal kubeconfig:

1. The user logs into Portainer.
2. Clicks their **username** in the top-right corner.
3. Selects **My account**.
4. Scrolls to the **Kubeconfig** section.
5. Selects the environment they want access to.
6. Clicks **Download kubeconfig**.

The downloaded file is pre-configured with:
- The Portainer API endpoint as the Kubernetes API server URL
- A unique, scoped access token
- Namespace restrictions based on their Portainer permissions

## Step 4: Using the Downloaded Kubeconfig

```bash
# Move kubeconfig to the default location

mv ~/Downloads/portainer-kubeconfig.yaml ~/.kube/config

# Or use it alongside an existing kubeconfig
export KUBECONFIG=~/.kube/config:~/Downloads/portainer-kubeconfig.yaml

# Merge kubeconfigs
kubectl config view --flatten > ~/.kube/merged-config
mv ~/.kube/merged-config ~/.kube/config

# Verify access
kubectl config get-contexts

# Switch to the Portainer context
kubectl config use-context portainer-production

# Test connectivity
kubectl get namespaces
kubectl get pods -n your-namespace
```

## Step 5: Verify RBAC Scoping

Each user's kubeconfig includes credentials that map to their Portainer permissions:

```bash
# Check what the user can access
kubectl auth can-i list pods -n production
kubectl auth can-i create deployments -n production
kubectl auth can-i delete namespaces  # Should be denied for non-admins

# View current context
kubectl config current-context

# View cluster info
kubectl cluster-info
```

## Step 6: Set Kubeconfig Expiry (Admin)

For security, configure token expiry to force periodic re-authentication:

1. In environment settings, find the **Kubeconfig expiry** option.
2. Set a duration: `4h`, `8h`, `24h`, or `7d`.
3. When tokens expire, users must re-download their kubeconfig from Portainer.

## Using the KubeShell Alternative

For users who don't need local kubectl access, Portainer provides a browser-based KubeShell:

1. In the Kubernetes environment, click **KubeShell** in the sidebar.
2. A terminal opens with kubectl pre-authenticated to the cluster.
3. Access is automatically scoped to the user's namespace permissions.

```bash
# KubeShell example - already authenticated
kubectl get pods -n my-namespace
kubectl logs deployment/myapp -n my-namespace --tail=50
kubectl rollout status deployment/myapp -n my-namespace
```

## Troubleshooting

```bash
# If kubectl reports unauthorized
kubectl config view  # Check token and server URL

# Test direct API connectivity
curl -k https://portainer.example.com/api/endpoints/1/kubernetes/version

# Regenerate kubeconfig by re-downloading from Portainer
```

## Conclusion

Portainer's kubectl proxy feature provides a secure and convenient way for teams to use the Kubernetes CLI with scoped access. Admins maintain control through Portainer's RBAC and namespace assignments, while developers get familiar kubectl workflows. Use token expiry to enforce security hygiene and the built-in KubeShell for quick ad-hoc commands.
