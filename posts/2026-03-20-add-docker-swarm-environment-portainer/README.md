# How to Add a Docker Swarm Environment to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Environments, Configuration, DevOps

Description: Connect an existing Docker Swarm cluster to Portainer for centralized management of Swarm services and stacks.

---

Adding environments to Portainer allows centralized management of containers across different infrastructure types. Each environment type has specific connection requirements.

## Prerequisites

- Portainer running and accessible
- Target environment accessible from the Portainer server
- Appropriate credentials or connection details

## Adding the Environment via the UI

1. Log in to Portainer as an administrator
2. Navigate to **Environments** in the left sidebar
3. Click **Add environment**
4. Select the appropriate environment type
5. Fill in the connection details
6. Click **Connect**

## Adding via API

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Add environment via API

curl -X POST \
  https://localhost:9443/api/endpoints \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "my-environment",
    "EndpointCreationType": 1,
    "URL": "unix:///var/run/docker.sock"
  }' \
  --insecure

# List all environments
curl -s https://localhost:9443/api/endpoints \
  -H "Authorization: Bearer $TOKEN" \
  --insecure | python3 -c "
import sys, json
envs = json.load(sys.stdin)
for e in envs:
    print(f'  ID: {e[\"Id\"]}, Name: {e[\"Name\"]}, Type: {e.get(\"Type\",\"?\")}')
"
```

## Environment Types Reference

| Type | Value | Description |
|------|-------|-------------|
| Docker standalone (socket) | 1 | Local Docker via socket |
| Agent | 2 | Remote via Portainer Agent |
| Azure ACI | 3 | Azure Container Instances |
| Docker standalone (API) | 4 | Remote Docker via TCP |
| Kubernetes | 7 | K8s via agent or kubeconfig |

## Verify the Connection

After adding the environment, verify it shows as healthy:

```bash
# Check environment status
curl -s https://localhost:9443/api/endpoints \
  -H "Authorization: Bearer $TOKEN" \
  --insecure | python3 -c "
import sys, json
envs = json.load(sys.stdin)
for e in envs:
    status = e.get('Status', 0)
    status_str = 'Online' if status == 1 else 'Offline'
    print(f'{e[\"Name\"]}: {status_str}')
"
```

---

*Monitor all your connected environments with [OneUptime](https://oneuptime.com) uptime monitoring.*
