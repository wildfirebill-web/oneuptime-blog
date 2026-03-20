# How to Use the Operator Role in Portainer for Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, RBAC, Operator Role, Business Edition

Description: Learn how to configure and use the Operator role in Portainer for Kubernetes environments to allow operations teams to manage workloads without admin privileges.

---

The Operator role in Portainer provides a middle ground between full administrator access and standard user restrictions. For Kubernetes environments, Operators can manage existing workloads without the ability to create new namespaces or modify cluster-level settings.

## Operator Role Capabilities in Kubernetes

Operators can:
- Start, stop, and restart pods and deployments
- View container logs and exec into pods
- Update existing deployments (image updates, replicas)
- View services, configmaps, and basic cluster state

Operators cannot:
- Create new namespaces
- Deploy new applications or stacks from scratch
- Manage Kubernetes secrets
- Modify cluster RBAC or node settings

## Assign Operator Role to a Team

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Get the Kubernetes environment ID

ENVS=$(curl -s https://localhost:9443/api/endpoints \
  -H "Authorization: Bearer $TOKEN" --insecure)
echo "Environments:"
echo "$ENVS" | python3 -c "
import sys, json
envs = json.load(sys.stdin)
for e in envs:
    print(f'  ID: {e[\"Id\"]}, Name: {e[\"Name\"]}, Type: {e.get(\"Type\",\"?\")}')
"

# Assign ops-team (team ID 4) as Operator (role ID 2) on environment 2
curl -X PUT \
  https://localhost:9443/api/endpoints/2/teamaccesspolicies \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"4": {"RoleID": 2}}' \
  --insecure

echo "Operator role assigned to ops-team on Kubernetes environment"
```

## Configure Namespace-Level Operator Access

For finer-grained control, assign the Operator role on specific namespaces:

1. Navigate to **Environments > [Kubernetes Env] > Namespaces**
2. Select the namespace
3. Go to **Access** tab
4. Assign the ops-team with Operator role on that namespace only

## What Operators See in Portainer for Kubernetes

- **Applications**: Can view and update deployments, scale replicas
- **Services**: Read-only view
- **Volumes**: Read-only view
- **Configurations**: View ConfigMaps, no Secrets access
- **Cluster**: Read-only cluster information

---

*Monitor your Kubernetes environments with [OneUptime](https://oneuptime.com) for full observability.*
