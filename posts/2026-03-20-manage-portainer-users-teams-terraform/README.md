# How to Manage Portainer Users and Teams with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Terraform, Users, Teams, Access Control

Description: Learn how to define and manage Portainer users, teams, and team memberships using Terraform for consistent, version-controlled access control.

## Overview

Managing users and teams through Terraform ensures your access control configuration is version-controlled, reviewable via pull requests, and reproducible. It eliminates manual user management in the UI.

## Users Resource

```hcl
# users.tf
variable "team_members" {
  description = "Map of team members to create"
  type = map(object({
    username = string
    role     = number  # 1 = admin, 2 = standard user
  }))
  default = {
    alice = { username = "alice.smith", role = 2 }
    bob   = { username = "bob.jones", role = 2 }
    carol = { username = "carol.admin", role = 1 }
  }
}

# Create users from the map
resource "portainer_user" "team" {
  for_each = var.team_members

  username = each.value.username
  password = var.default_user_password  # Should be changed on first login
  role     = each.value.role
}

variable "default_user_password" {
  type      = string
  sensitive = true
}
```

## Teams Resource

```hcl
# teams.tf
locals {
  teams = ["backend", "frontend", "devops", "data-engineering"]
}

resource "portainer_team" "teams" {
  for_each = toset(local.teams)
  name     = each.key
}

output "team_ids" {
  value = { for k, v in portainer_team.teams : k => v.id }
}
```

## Team Memberships

```hcl
# team_memberships.tf
locals {
  memberships = {
    alice_backend  = { team = "backend",  user = "alice", role = 2 }
    bob_backend    = { team = "backend",  user = "bob",   role = 1 }  # Team leader
    alice_devops   = { team = "devops",   user = "alice", role = 2 }
    carol_devops   = { team = "devops",   user = "carol", role = 1 }
  }
}

resource "portainer_team_membership" "memberships" {
  for_each = local.memberships

  team_id = portainer_team.teams[each.value.team].id
  user_id = portainer_user.team[each.value.user].id
  role    = each.value.role  # 1 = leader, 2 = member
}
```

## Granting Team Access to Environments

```hcl
# environment_access.tf
resource "portainer_environment_access" "backend_staging" {
  environment_id = portainer_environment.staging.id

  team_accesses = [
    {
      team_id = portainer_team.teams["backend"].id
      role_id = 2  # Standard user access
    }
  ]

  user_accesses = [
    {
      user_id = portainer_user.team["carol"].id
      role_id = 1  # Admin access for Carol in this environment
    }
  ]
}
```

## Complete Example with Outputs

```hcl
# outputs.tf
output "user_summary" {
  description = "Summary of created users"
  value = {
    for name, user in portainer_user.team :
    name => {
      id       = user.id
      username = user.username
      role     = user.role == 1 ? "admin" : "standard"
    }
  }
}

output "team_memberships" {
  description = "Team membership summary"
  value = {
    for key, membership in portainer_team_membership.memberships :
    key => {
      team = split("_", key)[1]
      user = split("_", key)[0]
      role = membership.role == 1 ? "leader" : "member"
    }
  }
}
```

## Managing Users at Scale

```bash
# Apply the Terraform configuration
terraform apply

# Add a new team member by updating variables
# Edit variables.tf to add the new member, then:
terraform apply  # Only creates/modifies changed resources

# Remove a user (will delete from Portainer)
# Remove from team_members map in variables.tf, then:
terraform apply
```

## CI/CD Integration for User Management

```yaml
# .github/workflows/portainer-users.yml
name: Manage Portainer Users

on:
  push:
    paths: ['terraform/users/**']
    branches: [main]

jobs:
  apply:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Apply
        working-directory: terraform/users
        env:
          TF_VAR_portainer_api_key: ${{ secrets.PORTAINER_API_KEY }}
          TF_VAR_default_user_password: ${{ secrets.DEFAULT_USER_PASSWORD }}
        run: |
          terraform init
          terraform apply -auto-approve
```

## Conclusion

Managing Portainer users and teams with Terraform enables access control as code. Team membership changes go through Git pull requests, ensuring peer review before access is granted or revoked. This is especially valuable for compliance-sensitive environments.
