# How to Override Team Roles for Individual Users in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, RBAC, Team, User Roles, Access Control, Override

Description: Override a team's default role for individual users to grant exceptions, such as giving one team member elevated access or restricting a specific user.

## Introduction

Sometimes you need to give one team member a different role than the rest of their team. For example, a tech lead might need Standard User access while their team has Helpdesk-only access. Or a contractor might need more restrictive access than their team. Portainer supports per-user access policies that override team-level policies.

## How Role Overrides Work

Portainer resolves a user's effective role using this priority order:
1. **Direct user access policy** (highest priority) - overrides team role
2. **Team access policy** - applies to all team members
3. **Group access policy** - inherited by environments in the group

When both a user policy and team policy exist for the same environment, the **more permissive** role takes effect.

**Important**: You can only grant a higher role via direct user policy - you cannot restrict a user to less than what their team has. If you need to restrict a user, consider removing them from the team.

## Granting Elevated Access to One User

Scenario: The `qa-team` has Helpdesk access to production, but the QA lead needs Standard User access.

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Team 3 (QA team) has Helpdesk access to environment 1

# Add direct Standard User access for user 7 (QA lead) in environment 1

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/useraccesspolicies \
  -d '{
    "7": {"RoleId": 2}
  }'
# User 7 now has Standard User access even though their team only has Helpdesk
```

## Getting Current User Access Policies

```bash
ENDPOINT_ID=1

# View all user and team access policies for an environment
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/${ENDPOINT_ID}" \
  | python3 -c "
import sys, json
e = json.load(sys.stdin)

print('=== Team Access Policies ===')
for team_id, policy in (e.get('TeamAccessPolicies') or {}).items():
    print(f'  Team {team_id}: RoleId={policy.get(\"RoleId\")}')

print('=== User Access Policies ===')
for user_id, policy in (e.get('UserAccessPolicies') or {}).items():
    print(f'  User {user_id}: RoleId={policy.get(\"RoleId\")}')
"
```

## Adding Multiple User Overrides

```bash
# Give multiple users elevated access in one API call
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/useraccesspolicies \
  -d '{
    "7": {"RoleId": 2},
    "9": {"RoleId": 2},
    "11": {"RoleId": 1}
  }'
```

Note: This replaces ALL user access policies for the environment. Include all users you want to have direct policies.

## Removing a User Override

```bash
# To remove user 7's direct policy, submit the list without them
# Get current policies first, then submit without the user you want to remove

CURRENT=$(curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1" \
  | python3 -c "
import sys, json
e = json.load(sys.stdin)
policies = e.get('UserAccessPolicies', {})
# Remove user 7
policies.pop('7', None)
print(json.dumps(policies))
")

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/useraccesspolicies \
  -d "$CURRENT"
```

## Verifying Effective Access

To understand what access a user actually has:

```bash
# Login as the user and check what they see
USER_TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"qa-lead","password":"password"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Check accessible environments and roles
curl -s \
  -H "Authorization: Bearer $USER_TOKEN" \
  https://portainer.example.com/api/endpoints \
  | python3 -c "
import sys, json
envs = json.load(sys.stdin)
for env in envs:
    print(f'Environment: {env[\"Name\"]} - UserRole: {env.get(\"UserAccessPolicies\", {}).get(\"me\", \"team-inherited\")}')
"
```

## Alternative: Team Subsets

For complex scenarios with many exceptions, consider creating sub-teams:

```text
qa-team → all QA engineers → Helpdesk access to Production
qa-leads → QA team leaders → Standard User access to Production

(Alice is in both qa-team and qa-leads - her effective role is Standard User)
```

This avoids per-user exceptions and keeps the access model team-based.

## Conclusion

User-level access overrides in Portainer provide flexibility for exception cases without restructuring your team setup. The key rule is that direct user policies can only elevate access above what a team provides - to restrict a user below their team's level, you must restructure team membership. For many exceptions, consider whether creating more granular teams would be cleaner than maintaining individual overrides.
