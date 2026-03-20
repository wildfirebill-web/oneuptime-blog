# How to Automate User Management in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, User Management, Automation, RBAC, API, Terraform

Description: Automate user management in Rancher using the Rancher API, Terraform, and scripts to provision users, assign roles, and manage team access at scale without manual UI interactions.

## Introduction

Managing users manually in the Rancher UI does not scale for organizations with dozens of teams and hundreds of users. Automating user provisioning through the Rancher API or Terraform enables consistent, auditable access management that integrates with HR systems, SSO directories, and CI/CD pipelines.

## Step 1: Rancher API Authentication

```bash
# Generate an API key in Rancher UI:

# User Avatar > Account & API Keys > Create API Key

# Store credentials securely
export RANCHER_URL="https://rancher.company.com"
export RANCHER_TOKEN="token-xxxxx:yyyyyyyyyyyyyyy"

# Test authentication
curl -sk -H "Authorization: Bearer $RANCHER_TOKEN" \
  "$RANCHER_URL/v3/users" | jq '.data[].username'
```

## Step 2: Create Users via API

```bash
# Create a new local user
curl -sk -X POST \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "jsmith",
    "password": "SecurePassword123!",
    "name": "John Smith",
    "enabled": true
  }' \
  "$RANCHER_URL/v3/users"

# Batch user creation from CSV
while IFS=',' read -r username name email; do
  curl -sk -X POST \
    -H "Authorization: Bearer $RANCHER_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"username\": \"$username\", \"name\": \"$name\", \"enabled\": true}" \
    "$RANCHER_URL/v3/users"
  echo "Created user: $username"
done < users.csv
```

## Step 3: Assign Cluster Roles

```bash
# Get cluster ID
CLUSTER_ID=$(curl -sk -H "Authorization: Bearer $RANCHER_TOKEN" \
  "$RANCHER_URL/v3/clusters" | jq -r '.data[] | select(.name=="production") | .id')

# Get user ID
USER_ID=$(curl -sk -H "Authorization: Bearer $RANCHER_TOKEN" \
  "$RANCHER_URL/v3/users?username=jsmith" | jq -r '.data[0].id')

# Assign cluster-member role
curl -sk -X POST \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"clusterId\": \"$CLUSTER_ID\",
    \"roleTemplateId\": \"cluster-member\",
    \"userPrincipalId\": \"local://$USER_ID\"
  }" \
  "$RANCHER_URL/v3/clusterroletemplatebindings"
```

## Step 4: Automate with Terraform

```hcl
# user-management.tf
terraform {
  required_providers {
    rancher2 = {
      source  = "rancher/rancher2"
      version = "~> 4.0"
    }
  }
}

provider "rancher2" {
  api_url    = var.rancher_url
  token_key  = var.rancher_token
}

# Define users in a structured way
variable "users" {
  type = list(object({
    username = string
    name     = string
    email    = string
    role     = string
  }))
  default = [
    { username = "jsmith",  name = "John Smith",  email = "jsmith@co.com",  role = "developer" },
    { username = "awilson", name = "Alice Wilson", email = "awilson@co.com", role = "viewer" }
  ]
}

resource "rancher2_user" "users" {
  for_each = { for u in var.users : u.username => u }
  username = each.value.username
  name     = each.value.name
  enabled  = true
}

resource "rancher2_cluster_role_template_binding" "bindings" {
  for_each          = { for u in var.users : u.username => u }
  cluster_id        = data.rancher2_cluster.production.id
  role_template_id  = each.value.role
  user_id           = rancher2_user.users[each.key].id
}
```

## Step 5: Sync with Active Directory Groups

```bash
# Script to sync AD group members to Rancher project roles
AD_GROUPS_URL="https://graph.microsoft.com/v1.0/groups"

sync_ad_group_to_rancher() {
  local group_id=$1
  local rancher_project=$2
  local role=$3

  # Get group members from Azure AD
  members=$(curl -s -H "Authorization: Bearer $AD_TOKEN" \
    "$AD_GROUPS_URL/$group_id/members" | \
    jq -r '.value[].userPrincipalName')

  # Sync each member to Rancher project
  for email in $members; do
    # Find user in Rancher by email
    user_id=$(curl -sk -H "Authorization: Bearer $RANCHER_TOKEN" \
      "$RANCHER_URL/v3/users?email=$email" | jq -r '.data[0].id')

    if [ -n "$user_id" ]; then
      # Ensure project role binding exists
      curl -sk -X POST \
        -H "Authorization: Bearer $RANCHER_TOKEN" \
        -H "Content-Type: application/json" \
        -d "{\"projectId\": \"$rancher_project\", \"roleTemplateId\": \"$role\", \"userId\": \"$user_id\"}" \
        "$RANCHER_URL/v3/projectroletemplatebindings"
    fi
  done
}

# Run sync
sync_ad_group_to_rancher "k8s-devs-group-id" "production-project-id" "developer"
```

## Step 6: Off-boarding Automation

```bash
# Disable departed user
disable_user() {
  local username=$1
  local user_id=$(curl -sk -H "Authorization: Bearer $RANCHER_TOKEN" \
    "$RANCHER_URL/v3/users?username=$username" | jq -r '.data[0].id')

  curl -sk -X PUT \
    -H "Authorization: Bearer $RANCHER_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"enabled": false}' \
    "$RANCHER_URL/v3/users/$user_id"

  echo "Disabled user: $username"

  # Optionally remove all role bindings
  bindings=$(curl -sk -H "Authorization: Bearer $RANCHER_TOKEN" \
    "$RANCHER_URL/v3/clusterroletemplatebindings?userId=$user_id" | \
    jq -r '.data[].id')

  for binding in $bindings; do
    curl -sk -X DELETE \
      -H "Authorization: Bearer $RANCHER_TOKEN" \
      "$RANCHER_URL/v3/clusterroletemplatebindings/$binding"
  done
}
```

## Conclusion

Automating user management in Rancher via the API and Terraform eliminates manual provisioning errors, enables consistent RBAC enforcement, and integrates with existing HR and identity management workflows. The Rancher API supports full CRUD operations on users and role bindings, enabling pipelines that automatically onboard new employees, sync AD group changes, and off-board departures. Store user definitions in version control for audit trails.
