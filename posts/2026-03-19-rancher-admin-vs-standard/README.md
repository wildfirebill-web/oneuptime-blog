# How to Set Up Admin vs Standard User Roles in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, RBAC, Permission, Role, Security

Description: Learn how to properly configure and differentiate administrator and standard user roles in Rancher for secure platform management.

Rancher distinguishes between administrators who manage the platform and standard users who consume it. Getting this separation right is fundamental to a secure Rancher deployment. This guide explains the differences between admin and standard user roles, when to use each, and how to configure them for your organization.

## Prerequisites

- Rancher v2.7+ installation
- Access to the initial administrator account
- An external authentication provider configured
- A plan for who should have administrative access

## Understanding the Role Difference

**Administrator** is a global role that grants unrestricted access to the entire Rancher instance:

- Create, modify, and delete clusters
- Manage all users and their permissions
- Change global settings and authentication configuration
- Access all clusters, projects, and namespaces
- Install and manage catalog apps

**Standard User** is a global role with more limited platform access:

- Log in to Rancher
- Create new clusters (and become the owner of those clusters)
- Access clusters and projects where explicitly granted membership

The key difference is that administrators can see and manage everything, while standard users can only access resources they have been specifically granted.

## Step 1: Identify Who Needs Admin Access

Admin access should be limited to people who need to:

- Configure Rancher itself (authentication, settings, branding)
- Manage the Rancher installation and upgrades
- Create and manage all clusters across the organization
- Troubleshoot platform-wide issues
- Manage user accounts and global role assignments

Typically this is a small platform or infrastructure team. Everyone else should be a standard user with appropriate cluster and project roles.

## Step 2: Configure the Initial Admin

During Rancher's first setup, you create an initial admin account. Secure this account:

1. Log in with the initial admin credentials.
2. Go to **User Avatar > Account & API Keys**.
3. Change the password to a strong, unique value.
4. Set up two-factor authentication if your auth provider supports it.

Create a second admin account as a backup:

1. Go to **Users & Authentication > Users**.
2. Click **Create**.
3. Fill in the username, display name, and password.
4. Under **Global Permissions**, check **Administrator**.
5. Click **Create**.

## Step 3: Set Standard User as the Default Role

Configure new users to receive the Standard User role when they first log in:

1. Go to **Users & Authentication > Roles > Global**.
2. Find the **Standard User** role.
3. Click the three-dot menu and select **Edit**.
4. Check **Yes: Default role for new users**.
5. Click **Save**.

Alternatively, if you want new users to have no access by default (more restrictive):

1. Uncheck **Standard User** as a default.
2. Ensure only **User-Base** is the default.
3. Manually assign roles as needed.

## Step 4: Create a Restricted Standard User Role

The built-in Standard User role allows cluster creation, which you may want to restrict. Create a custom global role:

```yaml
apiVersion: management.cattle.io/v3
kind: GlobalRole
metadata:
  name: restricted-standard-user
spec:
  displayName: Restricted Standard User
  description: "Can log in and access assigned clusters/projects but cannot create clusters"
  newUserDefault: true
  rules:
    - apiGroups: ["management.cattle.io"]
      resources: ["preferences"]
      verbs: ["*"]
    - apiGroups: ["management.cattle.io"]
      resources: ["settings"]
      verbs: ["get", "list", "watch"]
    - apiGroups: ["management.cattle.io"]
      resources: ["clusters"]
      verbs: ["get", "list", "watch"]
    - apiGroups: ["management.cattle.io"]
      resources: ["catalogs", "templates", "templateversions"]
      verbs: ["get", "list", "watch"]
```

```bash
kubectl apply -f restricted-standard-user.yaml
```

Then remove **Standard User** as a default:

1. Go to **Users & Authentication > Roles > Global**.
2. Edit **Standard User** and uncheck the default option.

## Step 5: Assign Admin Role to Specific Users

When a new platform team member joins:

1. Go to **Users & Authentication > Users**.
2. Find the user (they should already exist from logging in via the auth provider).
3. Click the three-dot menu and select **Edit Config**.
4. Under **Global Permissions**, check **Administrator**.
5. Click **Save**.

Via the API:

```bash
curl -X POST 'https://<rancher-url>/v3/globalrolebindings' \
  -H 'Authorization: Bearer <api-token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "globalRoleId": "admin",
    "userId": "<user-id>"
  }'
```

## Step 6: Downgrade Admin to Standard User

When someone leaves the platform team:

1. Go to **Users & Authentication > Users**.
2. Find the user and click **Edit Config**.
3. Under **Global Permissions**, uncheck **Administrator**.
4. Check **Standard User** or your custom restricted role.
5. Click **Save**.

The change takes effect immediately. The user will lose access to platform management features on their next page load.

## Step 7: Create an Intermediate Platform Role

For users who need more than standard access but should not be full administrators, create a platform operator role:

```yaml
apiVersion: management.cattle.io/v3
kind: GlobalRole
metadata:
  name: platform-operator
spec:
  displayName: Platform Operator
  description: "Can manage clusters and catalogs but cannot manage users or global settings"
  rules:
    - apiGroups: ["management.cattle.io"]
      resources: ["clusters"]
      verbs: ["create", "get", "list", "watch", "update", "delete"]
    - apiGroups: ["management.cattle.io"]
      resources: ["nodepools", "nodes"]
      verbs: ["create", "get", "list", "watch", "update", "delete"]
    - apiGroups: ["provisioning.cattle.io"]
      resources: ["clusters"]
      verbs: ["create", "get", "list", "watch", "update", "delete"]
    - apiGroups: ["catalog.cattle.io"]
      resources: ["clusterrepos", "apps", "operations"]
      verbs: ["*"]
    - apiGroups: ["management.cattle.io"]
      resources: ["catalogs", "templates", "templateversions"]
      verbs: ["*"]
    - apiGroups: ["management.cattle.io"]
      resources: ["preferences", "settings"]
      verbs: ["get", "list", "watch"]
```

```bash
kubectl apply -f platform-operator.yaml
```

## Step 8: Audit Admin Access

Regularly check who has admin access:

```bash
# List all administrators

echo "=== Current Administrators ==="
kubectl get globalrolebindings -o json | \
  jq -r '.items[] | select(.globalRoleName == "admin") | "\(.userName)\t\(.metadata.creationTimestamp)"' | \
  column -t -s $'\t'

echo ""
echo "=== Standard Users ==="
kubectl get globalrolebindings -o json | \
  jq -r '.items[] | select(.globalRoleName == "user") | "\(.userName)"'

echo ""
echo "Total admins: $(kubectl get globalrolebindings -o json | jq '[.items[] | select(.globalRoleName == "admin")] | length')"
echo "Total standard users: $(kubectl get globalrolebindings -o json | jq '[.items[] | select(.globalRoleName == "user")] | length')"
```

## Step 9: Configure Role-Based UI Experience

Standard users see a simplified Rancher UI that shows only the clusters and projects they can access. Admins see everything including global settings, user management, and all clusters.

To improve the experience for standard users:

1. Organize clusters with meaningful names and descriptions.
2. Set up projects within clusters before adding standard users.
3. Assign cluster and project roles so users see their resources immediately after logging in.

## Best Practices

- **Minimize admin accounts**: Keep the number of administrators to the smallest possible group (typically 2-5 people).
- **Use a break-glass admin**: Maintain one local admin account with a strong password stored in a vault for emergency access.
- **Prefer standard user + scoped roles**: Give users the standard user global role and then add cluster/project roles for specific access.
- **Document admin access**: Maintain a record of who has admin access and why.
- **Review monthly**: Check the admin list monthly and remove access for anyone who no longer needs it.
- **Separate day-to-day accounts**: Even platform team members should use standard user accounts for daily work and only use admin accounts when performing administrative tasks.

## Conclusion

The distinction between admin and standard user roles is the foundation of Rancher's security model. By limiting admin access to a small platform team and using standard user roles combined with cluster and project roles for everyone else, you maintain clear security boundaries. Create intermediate roles for specific needs rather than granting full admin access, and audit your admin list regularly.
