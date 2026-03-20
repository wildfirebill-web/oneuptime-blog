# How to Set Up the Edge Administrator Role in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge, RBAC, Administrator, Business Edition

Description: Configure the Edge Administrator role in Portainer Business Edition to delegate management of edge environments to specific users.

---

The Edge Administrator role in Portainer BE allows designated users to manage edge environments and edge agent deployments without having full global administrator access. This is ideal for distributed teams managing remote sites.

## Edge Administrator Capabilities

Edge Administrators can:
- View and manage edge environments they're assigned to
- Generate edge agent deployment scripts
- Configure edge agent settings
- View edge environment status and health
- Manage the Edge waiting room

Edge Administrators cannot:
- Manage non-edge environments
- Modify global Portainer settings
- Manage other users or teams

## Assign Edge Administrator Role

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Promote user to Edge Administrator
# Role 3 is typically the Edge Admin role in Portainer
curl -X PUT \
  https://localhost:9443/api/users/8 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"Role": 3}' \
  --insecure

echo "User promoted to Edge Administrator"
```

## Assign Edge Administrator to Edge Groups

Restrict Edge Admins to specific edge groups:

1. Navigate to **Edge > Edge Groups**
2. Create or select an edge group (e.g., "Site-London")
3. Under **Access**, assign the edge admin user or their team
4. They can now manage only the environments in this group

## Create Edge Groups via API

```bash
# Create an edge group
curl -X POST \
  https://localhost:9443/api/edge_groups \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "site-london",
    "Dynamic": false,
    "Endpoints": [5, 6, 7]
  }' \
  --insecure

# Assign edge admin user (ID 8) as administrator on this group
curl -X PUT \
  https://localhost:9443/api/edge_groups/1/access \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"8": {"RoleID": 1}}' \
  --insecure
```

---

*Monitor your edge deployments with [OneUptime](https://oneuptime.com) for distributed infrastructure visibility.*
