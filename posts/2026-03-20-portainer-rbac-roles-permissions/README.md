# How to Understand Portainer RBAC Roles and Permissions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, RBAC, Role, Permission, Business Edition

Description: A comprehensive guide to Portainer's role-based access control system including all available roles, their permissions, and how to apply them.

---

Portainer Business Edition implements a robust RBAC system that controls what users can see and do within each environment. Understanding the role hierarchy is essential for secure multi-user deployments.

## Role Hierarchy

Portainer has two levels of roles:
1. **System-level roles**: Apply globally across all environments
2. **Environment-level roles**: Apply to a specific environment or group

## System-Level Roles

| Role | Description |
|------|-------------|
| **Administrator** | Full control over all environments, users, and settings |
| **Standard User** | Access limited by environment-level role assignments |

## Environment-Level Roles (BE)

| Role | Level | Description |
|------|-------|-------------|
| **Environment Administrator** | 1 (highest) | Full control within the environment |
| **Operator** | 2 | Manage containers/stacks but not create them |
| **Helpdesk** | 3 | Read-only access to containers and logs |
| **Standard User** | 4 | Create and manage own resources |
| **Read-Only Viewer** | 5 | View resources, no changes |

## Permission Matrix

| Action | Env Admin | Operator | Standard User | Helpdesk |
|--------|-----------|----------|---------------|----------|
| View containers | ✓ | ✓ | ✓ | ✓ |
| Start/stop containers | ✓ | ✓ | ✓ | ✗ |
| Create containers | ✓ | ✗ | ✓ | ✗ |
| Delete containers | ✓ | ✗ | Own only | ✗ |
| Manage networks | ✓ | ✗ | ✓ | ✗ |
| Manage volumes | ✓ | ✗ | ✓ | ✗ |
| Deploy stacks | ✓ | ✗ | ✓ | ✗ |
| Manage environment | ✓ | ✗ | ✗ | ✗ |
| View logs | ✓ | ✓ | ✓ | ✓ |
| Container console | ✓ | ✓ | ✓ | ✗ |

## Assign Environment Roles via API

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Assign a team to an environment with Operator role (role ID: 2)

# Environment ID: 1, Team ID: 3
curl -X PUT \
  https://localhost:9443/api/endpoints/1/teamaccesspolicies \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"3": {"RoleID": 2}}' \
  --insecure
```

## Role IDs Reference

| Role Name | Role ID |
|-----------|---------|
| Environment Admin | 1 |
| Operator | 2 |
| Helpdesk | 3 |
| Standard User | 4 |
| Read-Only | 5 |

---

*Complement your RBAC strategy with infrastructure monitoring from [OneUptime](https://oneuptime.com).*
