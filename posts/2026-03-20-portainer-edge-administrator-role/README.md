# How to Set Up the Edge Administrator Role in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge, Administrator, RBAC, Edge Computing

Description: Configure the Edge Administrator role in Portainer Business Edition to delegate edge environment management without granting full global administrator access.

## Introduction

The Edge Administrator role in Portainer Business Edition allows designated users to manage edge environments (remote sites, IoT devices, branch offices) without having full global administrator access. This enables distributed teams to manage their own edge deployments while the central platform team retains control over core environments.

## Prerequisites

- Portainer Business Edition
- Edge computing features enabled
- Edge environments or edge groups configured

## Edge Administrator Capabilities

An Edge Administrator can:

- View and manage all edge environments
- Assign and revoke edge environments from edge groups
- Deploy stacks and services to edge environments
- View edge agent status and connectivity
- Create and manage edge groups
- Use the edge jobs feature (run jobs across edge devices)
- Configure async vs. standard edge agents

An Edge Administrator **cannot**:
- Manage standard (non-edge) environments
- Create or manage users or teams
- Configure global authentication settings
- Access Portainer's global settings
- Manage non-edge registries or templates

## Assigning the Edge Administrator Role

The Edge Administrator is a global role (not environment-specific):

### Via Web UI

1. Go to **Settings** → **Users**
2. Click on the user
3. Set role to **Edge Administrator**
4. Save

### Via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Create a new Edge Administrator

curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/users \
  -d '{
    "username": "edge-ops-1",
    "password": "SecureEdgeP@ssword123",
    "role": 4
  }'
# role: 4 = Edge Administrator

# Promote existing user (ID: 5) to Edge Administrator
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/users/5 \
  -d '{
    "role": 4
  }'
```

## Typical Deployment Scenario

```text
Org Structure:
  Central IT Team → Global Administrators
  US Branch IT   → Edge Administrator (manages US edge environments)
  EU Branch IT   → Edge Administrator (manages EU edge environments)
  APAC Branch IT → Edge Administrator (manages APAC edge environments)
```

The branch IT teams can fully manage their edge deployments without needing to involve Central IT for every operation.

## Edge Groups for Access Delegation

Edge Administrators work best combined with Edge Groups:

```bash
# Create an edge group
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/edge_groups \
  -d '{
    "name": "US-Branch-Offices",
    "dynamic": false,
    "endpoints": [10, 11, 12, 13]
  }'

# Dynamic edge groups (based on tags)
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/edge_groups \
  -d '{
    "name": "US-East-Sites",
    "dynamic": true,
    "tagIds": [5, 6]
  }'
```

Edge Administrators can then manage environments within groups they have access to.

## Edge Administrator vs. Global Administrator

| Task | Edge Admin | Global Admin |
|------|-----------|-------------|
| Manage edge environments | ✓ | ✓ |
| Deploy to edge | ✓ | ✓ |
| Manage edge groups | ✓ | ✓ |
| Manage standard environments | ✗ | ✓ |
| Manage users | ✗ | ✓ |
| Configure auth | ✗ | ✓ |
| View global settings | ✗ | ✓ |

## Testing Edge Administrator Access

```bash
# Login as Edge Administrator
EDGE_ADMIN_TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"edge-ops-1","password":"SecureEdgeP@ssword123"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# List accessible edge environments
curl -s \
  -H "Authorization: Bearer $EDGE_ADMIN_TOKEN" \
  https://portainer.example.com/api/endpoints \
  | python3 -c "
import sys, json
envs = json.load(sys.stdin)
edge_envs = [e for e in envs if e.get('Type') in [4, 7, 8]]
print(f'Edge environments accessible: {len(edge_envs)}')
for env in edge_envs:
    print(f'  - {env[\"Name\"]} (ID: {env[\"Id\"]})')
"

# Try to access standard environments (should be empty or restricted)
curl -s \
  -H "Authorization: Bearer $EDGE_ADMIN_TOKEN" \
  https://portainer.example.com/api/users \
  | python3 -m json.tool  # Should return 403 Forbidden
```

## Conclusion

The Edge Administrator role is the right tool for organizations with distributed edge deployments managed by separate teams. It provides full edge management capabilities while maintaining separation from central infrastructure. This role is especially valuable in OT/IoT environments, retail chains, or any organization with remote sites that have their own IT staff.
