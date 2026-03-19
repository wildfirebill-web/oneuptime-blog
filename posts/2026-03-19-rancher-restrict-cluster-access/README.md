# How to Restrict User Access to Specific Clusters in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, RBAC, Permissions, Security, Roles

Description: Learn how to limit users to only the clusters they need in Rancher by configuring cluster-level role bindings and removing broad access.

When managing multiple Kubernetes clusters through Rancher, you often need to ensure that users can only access specific clusters. A developer working on the staging environment should not see or interact with production clusters. This guide explains how to restrict user access so that each user or team only sees the clusters they are authorized to use.

## Prerequisites

- Rancher v2.7+ with administrator access
- Multiple downstream clusters managed by Rancher
- An external authentication provider configured
- Users or groups that need scoped access

## How Cluster Access Works in Rancher

Users can access a cluster in Rancher only if one of the following is true:

1. They have the **Administrator** global role (which grants access to everything).
2. They have a **Cluster Role Template Binding** for that specific cluster.
3. They have a **Project Role Template Binding** within a project on that cluster.

If none of these conditions are met, the user cannot see or interact with the cluster.

## Step 1: Remove Default Cluster Access

Rancher may assign default cluster roles to new users. Remove this behavior:

1. Go to **Global Settings** (hamburger menu > **Global Settings**).
2. Find the `cluster-default-role` setting.
3. Edit it and set the value to an empty string.
4. Click **Save**.

Via kubectl:

```bash
kubectl patch settings cluster-default-role -p '{"value": ""}'
```

This ensures that new users do not automatically gain access to any cluster.

## Step 2: Remove the Standard User Global Role as Default

The **Standard User** global role allows users to create new clusters and view existing ones. Change the default:

1. Go to **Users & Authentication > Roles > Global**.
2. Find **Standard User**.
3. Click the three-dot menu and select **Edit**.
4. Uncheck **Yes: Default role for new users**.
5. Click **Save**.
6. Ensure **User-Base** is the only default global role.

## Step 3: Grant Access to a Specific Cluster

Now grant access to individual clusters on a per-user or per-group basis.

**Via the UI:**

1. Navigate to the target cluster by clicking it in the cluster list.
2. Go to **Cluster > Cluster Members**.
3. Click **Add**.
4. Search for the user or group.
5. Select the appropriate cluster role (Cluster Member, Read-Only, or a custom role).
6. Click **Create**.

The user will only see this specific cluster in their Rancher dashboard.

**Via kubectl:**

```yaml
apiVersion: management.cattle.io/v3
kind: ClusterRoleTemplateBinding
metadata:
  generateName: crtb-
  namespace: c-m-xxxxx  # cluster ID
spec:
  clusterName: c-m-xxxxx
  roleTemplateId: cluster-member
  userPrincipalId: "local://u-xxxxx"
```

```bash
kubectl apply -f cluster-access.yaml
```

## Step 4: Restrict Access Using Groups

Map your organizational structure to cluster access using groups:

```bash
# Grant the dev-team group access to the development cluster
curl -X POST 'https://<rancher-url>/v3/clusterroletemplatebindings' \
  -H 'Authorization: Bearer <api-token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "clusterId": "c-m-dev01",
    "roleTemplateId": "cluster-member",
    "groupPrincipalId": "openldap_group://cn=dev-team,ou=groups,dc=example,dc=com"
  }'

# Grant the ops-team group access to the production cluster
curl -X POST 'https://<rancher-url>/v3/clusterroletemplatebindings' \
  -H 'Authorization: Bearer <api-token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "clusterId": "c-m-prod01",
    "roleTemplateId": "cluster-member",
    "groupPrincipalId": "openldap_group://cn=ops-team,ou=groups,dc=example,dc=com"
  }'
```

## Step 5: Manage Access with Terraform

For infrastructure-as-code workflows, manage cluster access through Terraform:

```hcl
# Development cluster - accessible by developers
resource "rancher2_cluster_role_template_binding" "dev_access" {
  name               = "dev-team-access"
  cluster_id         = rancher2_cluster.development.id
  role_template_id   = "cluster-member"
  group_principal_id = data.rancher2_principal.dev_team.id
}

# Staging cluster - accessible by developers and QA
resource "rancher2_cluster_role_template_binding" "staging_dev_access" {
  name               = "staging-dev-access"
  cluster_id         = rancher2_cluster.staging.id
  role_template_id   = "cluster-member"
  group_principal_id = data.rancher2_principal.dev_team.id
}

resource "rancher2_cluster_role_template_binding" "staging_qa_access" {
  name               = "staging-qa-access"
  cluster_id         = rancher2_cluster.staging.id
  role_template_id   = "cluster-member"
  group_principal_id = data.rancher2_principal.qa_team.id
}

# Production cluster - accessible only by ops
resource "rancher2_cluster_role_template_binding" "prod_access" {
  name               = "prod-ops-access"
  cluster_id         = rancher2_cluster.production.id
  role_template_id   = "cluster-member"
  group_principal_id = data.rancher2_principal.ops_team.id
}
```

## Step 6: Verify Access Restrictions

Log in as a restricted user and verify they can only see their assigned clusters:

1. Open an incognito browser window.
2. Log in with the restricted user's credentials.
3. The cluster list should show only the clusters the user has been granted access to.
4. Attempting to access another cluster's URL directly should return a forbidden error.

From the command line:

```bash
# As the restricted user, list accessible clusters
curl -s 'https://<rancher-url>/v3/clusters' \
  -H 'Authorization: Bearer <user-api-token>' | jq '.data[] | {name, id}'
```

The output should only contain clusters where the user has a role binding.

## Step 7: Revoke Cluster Access

To remove a user's access to a cluster:

**Via the UI:**
1. Navigate to the cluster.
2. Go to **Cluster > Cluster Members**.
3. Find the user or group.
4. Click the three-dot menu and select **Delete**.

**Via kubectl:**

```bash
# Find the binding
kubectl get clusterroletemplatebindings -n <cluster-id> | grep <username>

# Delete it
kubectl delete clusterroletemplatebinding <binding-name> -n <cluster-id>
```

## Step 8: Audit Cluster Access

Regularly audit who has access to each cluster:

```bash
#!/bin/bash
# List access for each cluster
for cluster in $(kubectl get clusters.management.cattle.io -o jsonpath='{.items[*].metadata.name}'); do
  echo "=== Cluster: $cluster ==="
  kubectl get clusterroletemplatebindings -n $cluster \
    -o custom-columns=USER:.userName,GROUP:.groupPrincipalName,ROLE:.roleTemplateId
  echo ""
done
```

## Best Practices

- **Default deny**: Ensure no default cluster roles are assigned to new users.
- **Use groups**: Map cluster access to identity provider groups for easier management.
- **Separate environments**: Never give development teams access to production clusters unless they have an operational role.
- **Read-only for most**: Give read-only cluster access to stakeholders who need visibility without the ability to make changes.
- **Regular audits**: Run monthly audits to identify and remove stale cluster access.
- **Use Terraform**: Manage access as code so that changes are tracked, reviewed, and reproducible.

## Conclusion

Restricting cluster access in Rancher is essential for maintaining security boundaries between environments and teams. By removing default access, assigning cluster roles explicitly, and using group-based bindings, you ensure that users can only interact with the clusters they are authorized to use. Combine this with regular audits to keep your access controls current.
