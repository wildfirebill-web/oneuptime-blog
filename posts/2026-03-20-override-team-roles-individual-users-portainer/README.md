# How to Override Team Roles for Individual Users in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, RBAC, Teams, User Override, Access Control

Description: Learn how to grant individual users different permissions than their team assignment in Portainer for exceptions to group-based access policies.

---

While team-based access control scales well, sometimes a specific user needs a different permission level than their team. Portainer allows per-user policy overrides that take precedence over team assignments.

## How User Policy Overrides Work

When both team and individual user policies exist for an environment, the **most permissive** role wins. This means:
- Team Operator + User Standard User → User gets Operator access
- Team Helpdesk + User Standard User → User gets Standard User access

To restrict a user below their team level, you cannot use additive overrides — you'd need to remove them from the team and assign individually.

## Assign Individual User Policy

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# View current access policies for environment 1
curl -s https://localhost:9443/api/endpoints/1 \
  -H "Authorization: Bearer $TOKEN" \
  --insecure | python3 -c "
import sys, json
e = json.load(sys.stdin)
print('Team Policies:', e.get('TeamAccessPolicies', {}))
print('User Policies:', e.get('UserAccessPolicies', {}))
"

# Grant user ID 7 Operator access (overriding their team's Standard User role)
curl -X PUT \
  https://localhost:9443/api/endpoints/1/useraccesspolicies \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"7": {"RoleID": 2}}' \
  --insecure

echo "User override policy applied"
```

## Remove an Individual Override

```bash
# Remove the user-specific override (user falls back to team role)
# Pass an empty object for the user ID to remove their individual policy
curl -X PUT \
  https://localhost:9443/api/endpoints/1/useraccesspolicies \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' \
  --insecure
# Note: This removes ALL user policies. Use cautiously.
```

## Practical Use Cases

**Temporary elevated access**: Grant an on-call engineer Operator access to a production environment for a limited period:

```bash
# Grant temporary operator access to on-call user
curl -X PUT https://localhost:9443/api/endpoints/2/useraccesspolicies \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"12": {"RoleID": 2}}' --insecure
echo "Temporary operator access granted to user 12"

# After incident resolves, remove the override
curl -X PUT https://localhost:9443/api/endpoints/2/useraccesspolicies \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' --insecure
echo "Temporary access removed"
```

---

*Track all access control changes with [OneUptime](https://oneuptime.com) audit logging and monitoring.*
