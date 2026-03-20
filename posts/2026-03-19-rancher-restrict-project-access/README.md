# How to Restrict User Access to Specific Projects in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, RBAC, Permissions, Projects, Security

Description: A practical guide to restricting user access to specific projects in Rancher so teams only see and manage their own namespaces and workloads.

Projects in Rancher group namespaces together and provide a natural boundary for team-based access control. By restricting users to specific projects, you ensure that developers only interact with their own team's workloads and cannot accidentally affect other teams' resources. This guide shows how to configure project-level access restrictions.

## Prerequisites

- Rancher v2.7+ with administrator or cluster owner access
- Multiple projects configured in your cluster
- Users or groups from an authentication provider

## How Project Access Works

A user can access a project in Rancher if:

1. They have a **Project Role Template Binding** for that specific project.
2. They have **Cluster Owner** access (which grants access to all projects in the cluster).
3. They have the **Administrator** global role.

Users with only **Cluster Member** access can view cluster-level resources and create new projects, but they cannot access existing projects unless explicitly added.

## Step 1: Plan Your Project Structure

Before configuring access, design your project layout:

```plaintext
Cluster: production
├── Project: frontend-team
│   ├── Namespace: frontend-prod
│   └── Namespace: frontend-staging
├── Project: backend-team
│   ├── Namespace: api-prod
│   └── Namespace: api-staging
├── Project: data-team
│   ├── Namespace: data-pipelines
│   └── Namespace: data-analytics
└── Project: shared-services
    ├── Namespace: monitoring
    └── Namespace: logging
```

Each team gets their own project with dedicated namespaces.

## Step 2: Create Projects

If the projects do not exist yet, create them:

1. Navigate to your cluster in the Rancher UI.
2. Go to **Cluster > Projects/Namespaces**.
3. Click **Create Project**.
4. Set the project name (for example, `frontend-team`).
5. Optionally add resource quotas and network isolation settings.
6. Click **Create**.

Repeat for each team's project.

## Step 3: Assign Users to Their Project Only

For each project, add only the team members who should have access:

1. Navigate to the project (click the project name under **Projects/Namespaces**).
2. Go to **Project > Members**.
3. Click **Add**.
4. Search for the user or group.
5. Select the appropriate role:
   - **Project Owner** for the team lead
   - **Project Member** for developers
   - **Read-Only** for stakeholders
6. Click **Create**.

**Example assignment via the API:**

```bash
# Add frontend-team group as Project Members to the frontend project

curl -X POST 'https://<rancher-url>/v3/projectroletemplatebindings' \
  -H 'Authorization: Bearer <api-token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "projectId": "c-m-xxxxx:p-frontend",
    "roleTemplateId": "project-member",
    "groupPrincipalId": "openldap_group://cn=frontend-team,ou=groups,dc=example,dc=com"
  }'
```

## Step 4: Remove Cluster Member Access

If users currently have **Cluster Member** access, they can see cluster-level resources and potentially create new projects. To restrict them to only project-level access:

1. Go to **Cluster > Cluster Members**.
2. Find users or groups that should only have project access.
3. Remove their cluster-level role by clicking the three-dot menu and selecting **Delete**.

The users will retain access to their specific projects through their project role bindings but will lose the ability to view cluster-level resources or create new projects.

## Step 5: Give Project-Only Access with Cluster Visibility

Sometimes users need minimal cluster visibility (to see the cluster in the Rancher dashboard) without broad cluster permissions. Create a minimal cluster role:

```yaml
apiVersion: management.cattle.io/v3
kind: RoleTemplate
metadata:
  name: cluster-viewer-minimal
spec:
  context: cluster
  displayName: Cluster Viewer (Minimal)
  rules:
    - apiGroups: [""]
      resources: ["namespaces"]
      verbs: ["get", "list", "watch"]
    - apiGroups: [""]
      resources: ["nodes"]
      verbs: ["get", "list", "watch"]
```

```bash
kubectl apply -f cluster-viewer-minimal.yaml
```

Assign this minimal cluster role alongside the project role:

1. Add the user to the cluster with the `cluster-viewer-minimal` role.
2. Add the user to their project with the appropriate project role.

## Step 6: Manage Project Access with Terraform

```hcl
# Create projects
resource "rancher2_project" "frontend" {
  name       = "frontend-team"
  cluster_id = rancher2_cluster.production.id
}

resource "rancher2_project" "backend" {
  name       = "backend-team"
  cluster_id = rancher2_cluster.production.id
}

# Assign teams to their respective projects only
resource "rancher2_project_role_template_binding" "frontend_devs" {
  name               = "frontend-devs"
  project_id         = rancher2_project.frontend.id
  role_template_id   = "project-member"
  group_principal_id = data.rancher2_principal.frontend_team.id
}

resource "rancher2_project_role_template_binding" "backend_devs" {
  name               = "backend-devs"
  project_id         = rancher2_project.backend.id
  role_template_id   = "project-member"
  group_principal_id = data.rancher2_principal.backend_team.id
}

# Give the frontend lead Project Owner on their project
resource "rancher2_project_role_template_binding" "frontend_lead" {
  name             = "frontend-lead"
  project_id       = rancher2_project.frontend.id
  role_template_id = "project-owner"
  user_id          = rancher2_user.frontend_lead.id
}
```

## Step 7: Verify the Restrictions

Log in as a frontend team member and verify:

```bash
# Should succeed (frontend namespace is in their project)
kubectl get pods -n frontend-prod

# Should fail (backend namespace is not in their project)
kubectl get pods -n api-prod

# Should fail (no cluster-level access)
kubectl get nodes
```

In the Rancher UI, the user should only see the `frontend-team` project and its namespaces.

## Step 8: Handle Cross-Project Visibility

Sometimes a user needs read-only access to another team's project for debugging or collaboration. Add them with the **Read-Only** project role:

1. Navigate to the other team's project.
2. Go to **Project > Members**.
3. Click **Add**.
4. Add the user with the **Read-Only** role.
5. Click **Create**.

The user can now view resources in both projects but can only modify resources in their own.

## Step 9: Audit Project Access

Regularly check who has access to each project:

```bash
#!/bin/bash
# Audit project access across all clusters
for ns in $(kubectl get projects.management.cattle.io --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'); do
  cluster=$(echo $ns | cut -d'/' -f1)
  project=$(echo $ns | cut -d'/' -f2)
  display=$(kubectl get projects.management.cattle.io $project -n $cluster -o jsonpath='{.spec.displayName}')
  echo "=== Project: $display (Cluster: $cluster) ==="
  kubectl get projectroletemplatebindings -n $cluster -o json | \
    jq -r ".items[] | select(.projectName == \"$cluster:$project\") | \"\(.userName // .groupPrincipalName) -> \(.roleTemplateId)\""
  echo ""
done
```

## Best Practices

- **One project per team**: Align projects with team boundaries for clean access separation.
- **Use groups**: Assign project roles to identity provider groups so that onboarding and offboarding is handled through your directory service.
- **Minimal cluster access**: If project-level access is sufficient, do not grant cluster-level roles.
- **Read-only for cross-team**: When teams need to see each other's resources, use read-only project roles.
- **Audit quarterly**: Review project membership to remove inactive users and adjust roles as teams change.

## Conclusion

Restricting users to specific projects in Rancher creates natural isolation between teams. Each team works in their own namespaces without visibility into or access to other teams' resources. By combining project-level role assignments with minimal cluster access, you build a secure multi-team environment where access is explicit, auditable, and aligned with your organization's structure.
