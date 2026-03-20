# How to Set Up the Helpdesk (Read-Only) Role in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, RBAC, Helpdesk, Read-Only, Access Control

Description: Configure the Helpdesk role in Portainer to grant read-only access to containers and logs for support teams without allowing any modifications.

---

This guide covers how to configure How to Set Up the Helpdesk (Read-Only) Role in Portainer in Portainer for secure multi-user environments.

## Overview

Proper access control in Portainer ensures that users only have the permissions they need to do their jobs, following the principle of least privilege.

## Configuration Steps

### Via the Portainer UI

1. Navigate to the relevant section in the Portainer admin interface
2. Select the environment, user, or team to configure
3. Assign the appropriate role
4. Save the configuration

### Via the API

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# List current access policies for environment ID 1
curl -s https://localhost:9443/api/endpoints/1 \
  -H "Authorization: Bearer $TOKEN" \
  --insecure | python3 -c "
import sys, json
e = json.load(sys.stdin)
print('Team Policies:', e.get('TeamAccessPolicies', {}))
print('User Policies:', e.get('UserAccessPolicies', {}))
"
```

## Role Reference

| Role ID | Role Name | Typical Use |
|---------|-----------|-------------|
| 1 | Environment Admin | Team leads, senior engineers |
| 2 | Operator | Operations team |
| 3 | Helpdesk | Support staff, auditors |
| 4 | Standard User | Developers |
| 5 | Read-Only | Viewers, stakeholders |

## Best Practices

- Assign roles based on job function, not seniority
- Use teams rather than individual user assignments for scalability
- Review access quarterly
- Use more restrictive roles for production environments
- Keep development environments more permissive

---

*Monitor your controlled infrastructure with [OneUptime](https://oneuptime.com) for full visibility.*
