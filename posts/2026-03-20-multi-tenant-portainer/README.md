# How to Set Up Multi-Tenant Container Management with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Multi-Tenant, Teams, Access Control, Enterprise

Description: Configure Portainer for multiple teams with isolated environments, role-based access control, and resource quotas so each tenant manages their own containers securely.

## Introduction

Multi-tenant container management allows different teams, departments, or customers to manage their own containers through a single Portainer instance without visibility into each other's workloads. Portainer Business Edition provides Teams, Role-Based Access Control (RBAC), and resource quotas. The CE edition offers basic user management with environment access control. This guide covers configuring Portainer for multi-tenancy at both the CE and Business tiers.

## Step 1: Create User Accounts for Each Tenant

```bash
PORTAINER_URL="https://portainer.example.com"
ADMIN_TOKEN="admin_api_token"

# Create a user for tenant A

curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/users" \
  -d '{
    "Username": "team-alpha-user",
    "Password": "SecurePassword123!",
    "Role": 2
  }'
# Role: 1=Admin, 2=Standard User

# Create user for tenant B
curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/users" \
  -d '{
    "Username": "team-beta-user",
    "Password": "AnotherPassword456!",
    "Role": 2
  }'
```

## Step 2: Create Teams (Portainer Business / EE)

```bash
# Create a team for each tenant
create_team() {
  local name=$1
  curl -s -X POST \
    -H "Authorization: Bearer $ADMIN_TOKEN" \
    -H "Content-Type: application/json" \
    "$PORTAINER_URL/api/teams" \
    -d "{\"Name\": \"$name\"}"
}

create_team "Team Alpha"
create_team "Team Beta"
create_team "Team Gamma"

# Get team IDs
curl -s \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  "$PORTAINER_URL/api/teams" | \
  jq '.[] | {id: .Id, name: .Name}'

# Add users to teams
# TEAM_ID=1 (Team Alpha), USER_ID=2 (team-alpha-user)
curl -s -X PUT \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/teams/1/memberships" \
  -d '{"Role": 1}'  # 1=Team Leader, 2=Team Member
```

## Step 3: Create Isolated Environments per Tenant

```bash
# Option 1: Separate Docker hosts per tenant (strongest isolation)
# Each team gets their own Portainer endpoint

# Register Team Alpha's Docker host
curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/endpoints" \
  -d '{
    "Name": "Team Alpha Production",
    "EndpointCreationType": 1,
    "URL": "tcp://team-alpha-docker.internal:9001",
    "TLS": true,
    "TLSSkipVerify": false
  }'

# Register Team Beta's Docker host
curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/endpoints" \
  -d '{
    "Name": "Team Beta Production",
    "EndpointCreationType": 1,
    "URL": "tcp://team-beta-docker.internal:9001"
  }'
```

## Step 4: Configure Access Control per Environment

```bash
# Grant Team Alpha access ONLY to their environment
# Environment ID: 1 (Team Alpha Production)
# Team ID: 1 (Team Alpha)

curl -s -X PUT \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/endpoints/1/access" \
  -d '{
    "AuthorizedTeams": [1],
    "AuthorizedUsers": [],
    "TeamAccessPolicies": {
      "1": {"RoleId": 1}
    }
  }'
# RoleId: 1=Endpoint Administrator, 2=Operator, 3=Helpdesk, 4=Standard User, 5=Read-Only

# Grant Team Beta access to their environment (ID: 2)
curl -s -X PUT \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/endpoints/2/access" \
  -d '{
    "AuthorizedTeams": [2],
    "TeamAccessPolicies": {
      "2": {"RoleId": 1}
    }
  }'

# Team Alpha now CANNOT see Team Beta's environment and vice versa
```

## Step 5: Network Isolation Between Tenants (Single Host)

If sharing a Docker host, use network namespaces:

```yaml
# Team Alpha's stack - isolated network
version: "3.8"

networks:
  team_alpha_net:
    name: team_alpha_net   # Predictable name for labeling
    driver: bridge
    labels:
      - "portainer.team=alpha"
    ipam:
      config:
        - subnet: 172.30.0.0/24  # Alpha's dedicated subnet

services:
  alpha_api:
    image: myapp/api:latest
    networks:
      - team_alpha_net
    labels:
      - "portainer.team=alpha"
```

```yaml
# Team Beta's stack - completely separate network
version: "3.8"

networks:
  team_beta_net:
    name: team_beta_net
    driver: bridge
    labels:
      - "portainer.team=beta"
    ipam:
      config:
        - subnet: 172.31.0.0/24  # Beta's dedicated subnet

services:
  beta_api:
    image: otherapp/api:latest
    networks:
      - team_beta_net
    labels:
      - "portainer.team=beta"
```

## Step 6: Implement Resource Quotas (Business Edition)

```bash
# Set resource quotas per environment (Business Edition)
curl -s -X PUT \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/endpoints/1" \
  -d '{
    "SecuritySettings": {
      "allowBindMountsForRegularUsers": false,
      "allowPrivilegedModeForRegularUsers": false,
      "allowHostNamespaceForRegularUsers": false,
      "allowDeviceMappingForRegularUsers": false,
      "allowSysctlSettingForRegularUsers": false,
      "allowContainerCapabilitiesForRegularUsers": false
    }
  }'

# Regular users cannot:
# - Mount host paths (only volumes)
# - Run privileged containers
# - Use host network/PID namespaces
# - Create custom devices
# - Modify kernel parameters
# - Add/drop capabilities
```

## Conclusion

Multi-tenancy in Portainer ranges from simple user-to-environment access restrictions (CE) to full team-based RBAC with security policies (Business Edition). The most important isolation principle is giving each tenant their own environment (Docker host or Swarm cluster) - this provides complete separation at the infrastructure level. When sharing a host, enforce network isolation through dedicated subnets per team and disable privileged mode for standard users. Regular audit of team memberships and environment access policies ensures that access rights stay aligned with organizational changes.
