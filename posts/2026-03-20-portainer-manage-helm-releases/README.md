# How to Manage Helm Releases in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Helm, DevOps, Management

Description: Learn how to view, upgrade, rollback, and delete Helm releases in Portainer to manage the full lifecycle of Helm-deployed applications on Kubernetes.

## Introduction

Once you deploy applications via Helm in Portainer, managing their lifecycle - upgrades, rollbacks, and uninstalls - is equally important. Portainer provides a Helm releases view that gives you a centralized overview of all deployed Helm charts and the tools to manage them without leaving the UI.

## Prerequisites

- Portainer CE or BE with a Kubernetes environment
- At least one Helm release deployed
- Admin or namespace-operator access

## Step 1: View All Helm Releases

1. Log into Portainer.
2. Select your **Kubernetes** environment.
3. Click **Helm** → **Releases** in the sidebar.

The releases list shows:
- **Release name**
- **Chart** and version
- **Namespace**
- **Status**: `deployed`, `failed`, `pending-install`, `superseded`
- **Updated**: Last modification timestamp
- **Revision**: Current deployment revision number

## Step 2: Inspect a Helm Release

Click a release name to see:
- **Chart metadata**: Chart version, app version, description
- **Current values**: The rendered values in use
- **Hooks**: Pre/post install hooks
- **Resources**: All Kubernetes resources created by this release

From the Portainer API:

```bash
TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' | jq -r '.jwt')

# List all Helm releases in a namespace

curl -s -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/helm?namespace=production" | jq .

# Get details of a specific release
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/helm/my-nginx?namespace=production" | jq .
```

## Step 3: Upgrade a Helm Release

### Via Portainer UI

1. In the Helm releases list, click the release you want to upgrade.
2. Click **Upgrade**.
3. Select a new chart version from the **Chart version** dropdown.
4. Modify values as needed in the YAML editor.
5. Click **Upgrade** to apply.

### Via KubeShell

```bash
# Upgrade with new values
helm upgrade my-nginx bitnami/nginx \
  --namespace production \
  --set replicaCount=3 \
  --reuse-values  # Preserve all other values

# Upgrade to a specific chart version
helm upgrade my-nginx bitnami/nginx \
  --namespace production \
  --version 16.0.0 \
  --reuse-values

# Upgrade with a values file
helm upgrade my-nginx bitnami/nginx \
  --namespace production \
  -f new-values.yaml

# Dry run to preview changes
helm upgrade my-nginx bitnami/nginx \
  --namespace production \
  --dry-run \
  --reuse-values
```

## Step 4: Rollback a Helm Release

If an upgrade causes issues, roll back to a previous revision:

### Via Portainer UI

1. Click the release name.
2. Scroll to the **History** section to see all revisions.
3. Select a previous revision.
4. Click **Rollback to this revision**.

### Via KubeShell

```bash
# View release history
helm history my-nginx -n production

# Output:
# REVISION  UPDATED                   STATUS      CHART        DESCRIPTION
# 1         2026-03-18 10:00:00 UTC   superseded  nginx-15.0.0  Install complete
# 2         2026-03-20 14:30:00 UTC   deployed    nginx-16.0.0  Upgrade complete

# Rollback to revision 1
helm rollback my-nginx 1 -n production

# Rollback to the immediately previous revision
helm rollback my-nginx -n production
```

## Step 5: Uninstall a Helm Release

### Via Portainer UI

1. In the releases list, check the checkbox next to the release.
2. Click **Remove**.
3. Confirm deletion.
4. Optionally check **Delete PVCs** to remove associated persistent storage.

### Via KubeShell

```bash
# Uninstall (removes all Kubernetes resources created by the release)
helm uninstall my-nginx -n production

# Uninstall and wait for resource deletion
helm uninstall my-nginx -n production --wait

# Uninstall but keep history (allows rollback later)
helm uninstall my-nginx -n production --keep-history
```

## Step 6: Monitor Release Health

```bash
# Check if all pods are running after a deployment
kubectl get pods -n production -l app.kubernetes.io/instance=my-nginx

# Watch rollout status
kubectl rollout status deployment/my-nginx -n production

# Get release notes
helm get notes my-nginx -n production

# Get all rendered YAML for the release
helm get manifest my-nginx -n production

# Check for failed hooks
helm get hooks my-nginx -n production
```

## Step 7: Managing Releases Across Namespaces

```bash
# List releases in all namespaces
helm list --all-namespaces

# List only failed releases
helm list --failed --all-namespaces

# Filter by chart name
helm list --filter 'nginx' --all-namespaces
```

## Conclusion

Managing Helm releases in Portainer gives you a visual interface for the full chart lifecycle, from initial deployment through upgrades and rollbacks to clean uninstallation. Use the Portainer UI for quick inspection and management, and the KubeShell for advanced operations like rollbacks and history review. Always test upgrades in staging before applying to production, and keep release history intact to enable rapid rollbacks when issues arise.
