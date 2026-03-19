# How to Assign Cluster Roles in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, RBAC, Permissions, Roles, Security

Description: A step-by-step guide to assigning cluster roles in Rancher for controlling user access at the cluster level.

Cluster roles in Rancher define what users can do within a specific Kubernetes cluster. Unlike global roles, which apply across the entire Rancher instance, cluster roles are scoped to individual clusters. This guide explains how to assign both built-in and custom cluster roles to users and groups.

## Prerequisites

- A running Rancher v2.7+ installation
- Administrator or cluster owner access
- At least one downstream cluster managed by Rancher
- Users or groups configured in your authentication provider

## Understanding Built-in Cluster Roles

Rancher provides several built-in cluster roles:

- **Cluster Owner**: Full administrative access to the cluster, including the ability to manage members, projects, and all resources.
- **Cluster Member**: Can create projects and view cluster-level resources but cannot modify cluster settings.
- **Read-Only**: Can view all cluster resources but cannot make any changes.

These roles serve as the foundation for cluster-level access control. You can also create custom cluster roles for more granular control.

## Step 1: Navigate to the Cluster

1. Log in to the Rancher UI.
2. Click the **hamburger menu** in the top-left corner.
3. Under **Explore Cluster**, select the cluster where you want to assign roles.

## Step 2: Access Cluster Members

1. In the cluster view, click **Cluster** in the left sidebar.
2. Select **Cluster Members** from the dropdown.

This page shows all users and groups currently assigned to the cluster along with their roles.

## Step 3: Add a User with a Cluster Role

1. Click **Add** in the top-right corner of the Cluster Members page.
2. In the **Member** field, search for the user by username, display name, or email address.
3. In the **Cluster Permissions** dropdown, select the desired role:
   - **Cluster Owner** for full cluster administration
   - **Cluster Member** for project creation and resource viewing
   - **Read-Only** for view-only access
   - Any **custom cluster roles** you have created
4. Click **Create** to save the assignment.

The user will now have the assigned permissions when they access this cluster through Rancher.

## Step 4: Assign Cluster Roles to Groups

If you have an external authentication provider configured (such as LDAP, Active Directory, or SAML), you can assign cluster roles to entire groups:

1. Go to **Cluster Members** and click **Add**.
2. Toggle from **User** to **Group** in the member selection.
3. Search for and select the group from your authentication provider.
4. Choose the cluster role to assign.
5. Click **Create**.

All members of that group will inherit the cluster role. This is the recommended approach for organizations with many users.

## Step 5: Assign Cluster Roles via the Rancher API

For automation, you can assign cluster roles using the Rancher API. First, get your API token from **User Avatar > Account & API Keys**.

```bash
# Assign a user to a cluster with the Cluster Member role
curl -X POST \
  'https://<rancher-url>/v3/clusterroletemplatebindings' \
  -H 'Authorization: Bearer <api-token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "clusterId": "c-m-xxxxx",
    "roleTemplateId": "cluster-member",
    "userPrincipalId": "local://<user-id>"
  }'
```

To find the cluster ID, navigate to the cluster in Rancher and check the URL, or use:

```bash
curl -s 'https://<rancher-url>/v3/clusters' \
  -H 'Authorization: Bearer <api-token>' | jq '.data[] | {id, name}'
```

## Step 6: Assign Cluster Roles via Terraform

If you manage Rancher with Terraform, use the `rancher2_cluster_role_template_binding` resource:

```hcl
resource "rancher2_cluster_role_template_binding" "dev_cluster_member" {
  name             = "dev-cluster-member"
  cluster_id       = rancher2_cluster.dev.id
  role_template_id = "cluster-member"
  user_id          = rancher2_user.developer.id
}

resource "rancher2_cluster_role_template_binding" "ops_cluster_owner" {
  name             = "ops-cluster-owner"
  cluster_id       = rancher2_cluster.production.id
  role_template_id = "cluster-owner"
  group_principal_id = "openldap_group://cn=ops,ou=groups,dc=example,dc=com"
}
```

Apply the configuration:

```bash
terraform plan
terraform apply
```

## Step 7: Modify an Existing Cluster Role Assignment

To change a user's cluster role:

1. Go to **Cluster Members** in the target cluster.
2. Find the user or group in the list.
3. Click the **three-dot menu** on the right side of their row.
4. Select **Edit**.
5. Change the **Cluster Permissions** to the new role.
6. Click **Save**.

Note that changing a role takes effect immediately. The user does not need to log out and log back in.

## Step 8: Remove a Cluster Role Assignment

To remove a user from a cluster:

1. Go to **Cluster Members**.
2. Find the user or group.
3. Click the **three-dot menu**.
4. Select **Delete**.
5. Confirm the deletion.

The user will immediately lose access to the cluster through Rancher.

## Step 9: Assign Multiple Cluster Roles

A user can have multiple cluster roles assigned simultaneously. The effective permissions are the union of all assigned roles. For example, if a user has both a custom read-only role and a custom role that allows managing deployments, they will be able to view everything and manage deployments.

To assign multiple roles:

1. Go to **Cluster Members** and click **Add**.
2. Select the user.
3. Select the first role.
4. Click **Create**.
5. Repeat the process to add additional roles for the same user.

## Verifying Cluster Role Assignments

After assigning roles, verify they work correctly:

```bash
# Log in as the assigned user and check permissions
kubectl auth can-i list pods --namespace default
kubectl auth can-i create deployments --namespace default
kubectl auth can-i delete nodes
```

You can also impersonate the user from an admin account:

```bash
kubectl auth can-i list pods --as=<username> --namespace default
```

## Best Practices

- **Use groups over individual users**: Assigning roles to groups from your identity provider is easier to manage and audit.
- **Start with Cluster Member**: Most users only need Cluster Member access. Grant Cluster Owner sparingly.
- **Document role assignments**: Keep records of who has access to which clusters and why.
- **Review regularly**: Audit cluster role assignments quarterly to remove stale access.
- **Use automation**: Manage role assignments through Terraform or the API for consistency across environments.

## Troubleshooting

If a user cannot access a cluster after role assignment:

1. Verify the role binding exists: Go to **Cluster Members** and confirm the user appears.
2. Check the authentication provider: Ensure the user's account is active.
3. Review cluster health: The downstream cluster must be in an **Active** state for role assignments to take effect.
4. Check for conflicting policies: Pod security policies or OPA/Gatekeeper rules might restrict actions that the role allows.

## Conclusion

Assigning cluster roles in Rancher is the primary mechanism for controlling who can do what within your Kubernetes clusters. By using built-in roles for common scenarios and custom roles for specialized needs, you can implement precise access controls. Prefer group-based assignments for scalability and use automation tools like Terraform for consistency across your infrastructure.
