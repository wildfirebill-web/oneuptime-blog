# How to Configure Restricted Admin Role in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, RBAC, Permissions, Role, Security

Description: Learn how to configure and use Rancher's restricted admin role to delegate administrative tasks without granting full platform access.

Rancher's restricted admin role provides a middle ground between the full administrator role and the standard user role. Restricted admins can manage most aspects of Rancher but are prevented from accessing the local (management) cluster and certain global settings. This guide explains how to set up and use the restricted admin feature.

## Prerequisites

- Rancher v2.7+ installation
- Full administrator access to enable the feature
- Understanding of which users need elevated but not unrestricted access

## Understanding the Restricted Admin Role

The restricted admin role was introduced to address a common scenario: organizations need multiple people who can manage clusters, users, and settings, but giving everyone full admin access creates unnecessary risk.

A restricted admin can:

- Manage downstream clusters (create, edit, delete)
- Manage users and assign roles
- Configure authentication providers
- Manage Helm chart repositories and apps
- View and modify most global settings

A restricted admin cannot:

- Access the local (management) cluster where Rancher itself runs
- Modify critical infrastructure settings
- Access the Rancher server's underlying Kubernetes resources

## Step 1: Enable the Restricted Admin Feature

The restricted admin feature is controlled by a global setting:

**Via the UI:**

1. Log in as a full administrator.
2. Go to **Global Settings** (hamburger menu > **Global Settings**).
3. Find the setting `restricted-admin`.
4. Click **Edit** (or the pencil icon).
5. Set the value to `true`.
6. Click **Save**.

**Via kubectl:**

```bash
kubectl patch settings restricted-admin -p '{"value": "true"}'
```

**Verify the setting:**

```bash
kubectl get settings restricted-admin -o jsonpath='{.value}'
```

## Step 2: Assign the Restricted Admin Role

Once enabled, the restricted admin role appears in the global roles list.

**Via the UI:**

1. Go to **Users & Authentication > Users**.
2. Find the user who should become a restricted admin.
3. Click the three-dot menu and select **Edit Config**.
4. Under **Global Permissions**, select **Restricted Administrator**.
5. Click **Save**.

**Via the API:**

```bash
# First, find the user ID

curl -s 'https://<rancher-url>/v3/users' \
  -H 'Authorization: Bearer <api-token>' | \
  jq -r '.data[] | "\(.id)\t\(.username)"' | column -t

# Assign the restricted admin role
curl -X POST 'https://<rancher-url>/v3/globalrolebindings' \
  -H 'Authorization: Bearer <api-token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "globalRoleId": "restricted-admin",
    "userId": "<user-id>"
  }'
```

## Step 3: Verify Restricted Admin Access

Log in as the restricted admin and verify the access boundaries.

**Should be accessible:**

1. Navigate to any downstream cluster - should work.
2. Go to **Users & Authentication** - should be accessible.
3. Create a new cluster - should work.
4. Add users to clusters and projects - should work.

**Should be restricted:**

1. Navigate to the **local** cluster - should be hidden or access denied.
2. Attempt to modify Rancher's infrastructure settings - should be denied.

**Via kubectl as the restricted admin:**

```bash
# Should work - downstream cluster access
kubectl get nodes --context=downstream-cluster

# Should fail - local cluster access
kubectl get nodes --context=local
```

## Step 4: Configure Local Cluster Access

The key feature of the restricted admin is that the local cluster (where Rancher runs) is hidden. However, you can fine-tune what restricted admins see:

**To completely hide the local cluster:**

This is the default behavior when restricted admin is enabled. The local cluster does not appear in the restricted admin's cluster list.

**To grant limited local cluster access to specific restricted admins:**

If certain restricted admins need read-only access to the local cluster for monitoring:

```yaml
apiVersion: management.cattle.io/v3
kind: ClusterRoleTemplateBinding
metadata:
  generateName: crtb-
  namespace: local
spec:
  clusterName: local
  roleTemplateId: cluster-member
  userPrincipalId: "local://u-xxxxx"
```

Use this sparingly, as it partially defeats the purpose of restricted admin.

## Step 5: Set Up a Restricted Admin Hierarchy

Create a structure where full admins manage the platform, restricted admins manage clusters, and standard users consume resources:

```plaintext
Full Administrators (2-3 people)
├── Manage Rancher installation
├── Access the local cluster
├── Configure global settings
└── Break-glass emergency access

Restricted Administrators (5-10 people)
├── Create and manage downstream clusters
├── Manage user accounts and roles
├── Configure authentication
└── Manage catalogs and apps

Standard Users (everyone else)
├── Access assigned clusters
├── Deploy workloads in assigned projects
└── View resources in their scope
```

## Step 6: Migrate From Full Admin to Restricted Admin

If you currently have too many full administrators, migrate them to restricted admin:

```bash
#!/bin/bash
# migrate-to-restricted-admin.sh

# List current admins
echo "Current full administrators:"
kubectl get globalrolebindings -o json | \
  jq -r '.items[] | select(.globalRoleName == "admin") | .userName'

# For each user to migrate (except the core platform team):
# 1. Remove the admin global role binding
# 2. Add the restricted-admin global role binding

USER_TO_MIGRATE="u-xxxxx"

# Find and delete the admin binding
ADMIN_BINDING=$(kubectl get globalrolebindings -o json | \
  jq -r ".items[] | select(.globalRoleName == \"admin\" and .userName == \"$USER_TO_MIGRATE\") | .metadata.name")

if [ -n "$ADMIN_BINDING" ]; then
  echo "Removing admin binding: $ADMIN_BINDING"
  kubectl delete globalrolebinding $ADMIN_BINDING

  echo "Creating restricted-admin binding"
  kubectl apply -f - <<EOF
apiVersion: management.cattle.io/v3
kind: GlobalRoleBinding
metadata:
  generateName: grb-
spec:
  globalRoleName: restricted-admin
  userName: $USER_TO_MIGRATE
EOF

  echo "Migration complete for $USER_TO_MIGRATE"
fi
```

## Step 7: Audit Restricted Admin Actions

Monitor what restricted admins do:

```bash
# Check audit logs for restricted admin actions
# Filter by users with the restricted-admin role

RESTRICTED_ADMINS=$(kubectl get globalrolebindings -o json | \
  jq -r '.items[] | select(.globalRoleName == "restricted-admin") | .userName')

echo "Restricted admin users: $RESTRICTED_ADMINS"

# Check for any attempts to access the local cluster
grep "local" /var/log/rancher/audit/audit.log | \
  jq -r 'select(.user.username as $u | env.RESTRICTED_ADMINS | contains($u)) | "\(.requestReceivedTimestamp) \(.user.username) \(.verb) \(.objectRef.resource)"'
```

## Step 8: Customize Restricted Admin Permissions

If the built-in restricted admin role does not exactly match your needs, create a custom global role that mirrors it with modifications:

```yaml
apiVersion: management.cattle.io/v3
kind: GlobalRole
metadata:
  name: custom-restricted-admin
spec:
  displayName: Custom Restricted Admin
  description: "Admin-like access without local cluster and without user management"
  rules:
    - apiGroups: ["management.cattle.io"]
      resources: ["clusters"]
      verbs: ["create", "get", "list", "watch", "update", "delete"]
    - apiGroups: ["provisioning.cattle.io"]
      resources: ["clusters"]
      verbs: ["create", "get", "list", "watch", "update", "delete"]
    - apiGroups: ["management.cattle.io"]
      resources: ["nodepools", "nodes", "clustertemplates"]
      verbs: ["*"]
    - apiGroups: ["catalog.cattle.io"]
      resources: ["clusterrepos", "apps", "operations"]
      verbs: ["*"]
    - apiGroups: ["management.cattle.io"]
      resources: ["settings"]
      verbs: ["get", "list", "watch"]
    - apiGroups: ["management.cattle.io"]
      resources: ["preferences"]
      verbs: ["*"]
```

This version removes user management permissions, limiting the role to cluster and catalog operations only.

## Best Practices

- **Start with restricted admin**: Default to restricted admin for elevated access and only grant full admin when absolutely necessary.
- **Keep full admins to 2-3**: Only the core platform team should have full administrator access.
- **Enable the feature early**: Turn on restricted admin before you have many admin users to avoid a disruptive migration later.
- **Document the boundary**: Make sure restricted admins understand what they can and cannot do.
- **Use with auth provider groups**: Assign the restricted admin role to an identity provider group for easier lifecycle management.
- **Audit the local cluster**: Monitor access attempts to the local cluster to detect any boundary violations.

## Conclusion

The restricted admin role in Rancher provides a secure way to delegate administrative responsibilities without exposing the management infrastructure. By enabling this feature, migrating excess full admins to restricted admin, and monitoring access patterns, you create a clear separation between platform management and cluster management. This reduces risk while still empowering teams to manage their Kubernetes infrastructure.
