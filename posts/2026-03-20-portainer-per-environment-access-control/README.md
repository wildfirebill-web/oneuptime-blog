# How to Configure Per-Environment Access Control in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Access Control, Environments, RBAC, Teams

Description: Set up granular access control for individual Portainer environments, controlling which users and teams can access each environment and with what role.

## Introduction

Per-environment access control is how you implement least-privilege access in Portainer. Instead of giving users broad access to all environments, you assign specific roles for each environment individually. This guide covers configuring environment-level access for users, teams, and the default access policy.

## Understanding Environment Access Architecture

Each Portainer environment has its own access policies:
- **User access policies**: Direct user-to-environment assignments
- **Team access policies**: Team-to-environment assignments

A user's effective role in an environment is determined by:
1. Their direct user access policy for the environment (if set)
2. Their team's access policy for the environment
3. Global admin always has access to everything

## Configuring Environment Access via UI

1. Go to **Environments** in the left menu
2. Click on the environment name
3. Click **Manage access** (top right)
4. Add users or teams and select their role

## Configuring Environment Access via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Set team access policies for environment 1
# Team 2 = DevOps (Standard User), Team 3 = QA (Standard User), Team 4 = Support (Helpdesk)
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/teamaccesspolicies \
  -d '{
    "2": {"RoleId": 2},
    "3": {"RoleId": 2},
    "4": {"RoleId": 1}
  }'

# Set individual user access policies for environment 1
# User 5 = alice (Standard User), User 6 = bob (Helpdesk)
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/useraccesspolicies \
  -d '{
    "5": {"RoleId": 2},
    "6": {"RoleId": 1}
  }'
```

## Access Control Design Patterns

### Pattern 1: Environment-Per-Project

```
Production Environment:
  - Platform team (Admin): Administrator
  - DevOps team: Standard User
  - Support team: Helpdesk

Staging Environment:
  - Platform team (Admin): Administrator
  - DevOps team: Standard User
  - QA team: Standard User
  - Support team: Standard User

Development Environment:
  - All developers: Standard User
  - Everyone else: Helpdesk
```

### Pattern 2: Environment-Per-Team

```
Backend Environment:
  - Backend team: Standard User
  - Everyone else: No access

Frontend Environment:
  - Frontend team: Standard User
  - Backend team: Helpdesk (view only)
  - Everyone else: No access
```

## Reviewing Current Access Policies

```bash
# Get all environment access policies
ENDPOINT_ID=1

# Team policies
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/${ENDPOINT_ID}" \
  | python3 -c "
import sys, json
e = json.load(sys.stdin)
team_policies = e.get('TeamAccessPolicies', {})
user_policies = e.get('UserAccessPolicies', {})
print('Team Access Policies:')
for team_id, policy in team_policies.items():
    print(f'  Team {team_id}: RoleId={policy.get(\"RoleId\")}')
print('User Access Policies:')
for user_id, policy in user_policies.items():
    print(f'  User {user_id}: RoleId={policy.get(\"RoleId\")}')
"
```

## Removing Access

```bash
# Remove a team from an environment by setting empty policies
# (Keep other teams, just remove one)

# Get current team access policies
CURRENT=$(curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1" \
  | python3 -c "import sys,json; print(json.dumps(json.load(sys.stdin).get('TeamAccessPolicies', {})))")

echo "Current policies: $CURRENT"

# Update without team ID 3
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/teamaccesspolicies \
  -d '{
    "2": {"RoleId": 2},
    "4": {"RoleId": 1}
  }'
  # Team 3 removed by not including it
```

## Bulk Access Configuration Script

```bash
#!/bin/bash
# configure-environment-access.sh

TOKEN="your-admin-token"
PORTAINER_URL="https://portainer.example.com"

# Environment access mapping: "endpoint_id:team_id:role_id"
ACCESS_CONFIG=(
  "1:2:2"   # Env 1, DevOps (team 2), Standard User
  "1:3:2"   # Env 1, QA (team 3), Standard User
  "1:4:1"   # Env 1, Support (team 4), Helpdesk
  "2:2:2"   # Env 2, DevOps, Standard User
  "2:4:1"   # Env 2, Support, Helpdesk
)

# Build access policies per environment
declare -A ENV_POLICIES

for config in "${ACCESS_CONFIG[@]}"; do
  IFS=':' read -r endpoint_id team_id role_id <<< "$config"

  if [[ -z "${ENV_POLICIES[$endpoint_id]}" ]]; then
    ENV_POLICIES[$endpoint_id]="{\"${team_id}\":{\"RoleId\":${role_id}}"
  else
    ENV_POLICIES[$endpoint_id]="${ENV_POLICIES[$endpoint_id]},\"${team_id}\":{\"RoleId\":${role_id}}"
  fi
done

# Apply policies
for endpoint_id in "${!ENV_POLICIES[@]}"; do
  POLICY="${ENV_POLICIES[$endpoint_id]}}"
  echo "Setting access for endpoint $endpoint_id: $POLICY"
  curl -s -X PUT \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "${PORTAINER_URL}/api/endpoints/${endpoint_id}/teamaccesspolicies" \
    -d "$POLICY"
done
```

## Conclusion

Per-environment access control is the foundation of Portainer's least-privilege model. By assigning specific roles to specific teams for each environment, you create a fine-grained permission structure that reflects your organization's actual needs. Audit these policies regularly to remove access that's no longer needed and to ensure new environments are properly secured from day one.
