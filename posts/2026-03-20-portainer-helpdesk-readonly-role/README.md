# How to Set Up the Helpdesk (Read-Only) Role in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Helpdesk, Read-Only, RBAC, Access Control, Support

Description: Configure the Helpdesk read-only role in Portainer for support teams who need visibility into container environments without management capabilities.

## Introduction

The Helpdesk role (also called Read-Only in some documentation) provides visibility into Portainer environments without the ability to make changes. Support teams, auditors, and junior staff can use this role to troubleshoot issues and gather information without the risk of accidentally modifying production environments.

## Helpdesk Role Capabilities

**What Helpdesk Users CAN do:**
- View container list and status (running, stopped, exited)
- View container details (image, ports, env vars, mounts)
- View container logs (read-only)
- View images, volumes, and networks
- View stack configurations
- View service details in Swarm environments
- View Kubernetes deployments and pod status (read-only)
- View resource usage and statistics

**What Helpdesk Users CANNOT do:**
- Start, stop, restart, or remove containers
- Deploy new containers or stacks
- Execute commands in containers (no terminal access)
- Modify any configuration
- Create or delete resources of any kind

## Assigning the Helpdesk Role

### Via Web UI

1. Navigate to **Environments** → select the environment
2. Click **Manage access**
3. Add the team or user
4. Set role to **Helpdesk**

### Via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Assign team 4 (support team) to environment 1 with Helpdesk role
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/teamaccesspolicies \
  -d '{
    "4": {"RoleId": 3}
  }'
# RoleId 3 = Helpdesk (Read-Only)

# Assign individual user (ID: 8) to environment 1 with Helpdesk role
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/useraccesspolicies \
  -d '{
    "8": {"RoleId": 3}
  }'
```

## Creating a Helpdesk Team and Assigning to Multiple Environments

```bash
# Step 1: Create the support team
TEAM_RESPONSE=$(curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/teams \
  -d '{"name": "support-team"}')

TEAM_ID=$(echo $TEAM_RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin)['Id'])")
echo "Support team ID: $TEAM_ID"

# Step 2: Add users to the support team
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/teams/${TEAM_ID}/memberships \
  -d "{\"userID\": 8, \"teamID\": ${TEAM_ID}, \"role\": 2}"

# Step 3: Grant Helpdesk access to all environments
ENDPOINTS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/endpoints \
  | python3 -c "import sys,json; [print(e['Id']) for e in json.load(sys.stdin)]")

for endpoint_id in $ENDPOINTS; do
  echo "Assigning helpdesk access to environment $endpoint_id"
  curl -s -X PUT \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "https://portainer.example.com/api/endpoints/${endpoint_id}/teamaccesspolicies" \
    -d "{\"${TEAM_ID}\": {\"RoleId\": 3}}"
done
```

## Helpdesk Use Case: Troubleshooting Workflow

A typical helpdesk investigation using read-only access:

```
1. User reports "Website is down"
2. Helpdesk logs into Portainer
3. Navigates to production environment
4. Views container list → sees nginx container is "Exited"
5. Opens container → views logs → sees "cannot open configuration file"
6. Reports to DevOps: "nginx container exited, configuration file error"
7. DevOps restarts the container with correct config
```

The helpdesk never needs to make changes — they gather information and escalate.

## Viewing Container Logs as Helpdesk

```bash
# Helpdesk user can view logs via Portainer UI
# They can also use the API with their own token

HELPDESK_TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"helpdesk-user","password":"password"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# View container logs (read-only operation)
curl -s \
  -H "Authorization: Bearer $HELPDESK_TOKEN" \
  "https://portainer.example.com/api/endpoints/1/docker/containers/nginx/logs?tail=100&stdout=true&stderr=true"
```

## Conclusion

The Helpdesk role provides the minimum access needed for support teams to do their job effectively — they can see everything but change nothing. Grant this role broadly to support staff and read-only stakeholders, then assign the minimum necessary write access only to those who need it. Combined with good logging and monitoring, helpdesk users can resolve most Level 1 issues without escalation.
