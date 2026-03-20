# How to Isolate Tenants Using Portainer Teams and Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Multi-Tenant, Teams, Isolation, Access Control

Description: Use Portainer's Teams feature and environment access control to create strong boundaries between tenant workloads, preventing cross-team visibility and resource access.

## Introduction

Portainer Teams provide a grouping mechanism for users, and environment access control determines which teams can see and manage which Docker environments. By mapping each tenant to a team and each team to specific environments, you create administrative boundaries that prevent one tenant from viewing or modifying another tenant's containers. This guide covers the complete tenant isolation setup workflow.

## Step 1: Plan Your Tenant Structure

```
Multi-tenant architecture:

Portainer Server
├── Admin User (full access)
│
├── Team: Alpha Corp
│   ├── User: alice@alpha.com (Team Leader)
│   ├── User: bob@alpha.com (Team Member)
│   └── Environment: alpha-production (Portainer Agent on host-1)
│       └── Environment: alpha-staging (Portainer Agent on host-2)
│
├── Team: Beta Inc
│   ├── User: charlie@beta.com (Team Leader)
│   └── Environment: beta-production (Portainer Agent on host-3)
│
└── Team: Internal Dev
    ├── User: dev-team@company.com
    └── Environment: shared-dev (Docker host-4)
```

## Step 2: Create Teams and Assign Users

```bash
PORTAINER_URL="https://portainer.example.com"
ADMIN_TOKEN="admin_token"

# Create Team: Alpha Corp
ALPHA_TEAM=$(curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/teams" \
  -d '{"Name": "Alpha Corp"}' | jq -r '.Id')

echo "Alpha Corp team ID: $ALPHA_TEAM"

# Create user Alice (Team Leader)
ALICE_ID=$(curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/users" \
  -d '{
    "Username": "alice",
    "Password": "Alice@Secure123!",
    "Role": 2
  }' | jq -r '.Id')

# Add Alice to Alpha Corp as Team Leader (role 1)
curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/teams/$ALPHA_TEAM/memberships" \
  -d "{\"UserID\": $ALICE_ID, \"Role\": 1}"

# Add Bob to Alpha Corp as Team Member (role 2)
BOB_ID=$(curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/users" \
  -d '{"Username": "bob", "Password": "Bob@Secure123!", "Role": 2}' | \
  jq -r '.Id')

curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/teams/$ALPHA_TEAM/memberships" \
  -d "{\"UserID\": $BOB_ID, \"Role\": 2}"
```

## Step 3: Create and Restrict Environments per Tenant

```bash
# Register Alpha Corp's production environment
ALPHA_ENV_ID=$(curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/endpoints" \
  -d '{
    "Name": "alpha-production",
    "EndpointCreationType": 2,
    "URL": "tcp://alpha-host.internal:9001",
    "PublicURL": "alpha-host.internal"
  }' | jq -r '.Id')

# Restrict environment to Alpha Corp team ONLY
curl -s -X PUT \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/endpoints/$ALPHA_ENV_ID/access" \
  -d "{
    \"AuthorizedTeams\": [$ALPHA_TEAM],
    \"AuthorizedUsers\": [],
    \"TeamAccessPolicies\": {
      \"$ALPHA_TEAM\": {\"RoleId\": 1}
    },
    \"UserAccessPolicies\": {}
  }"

# Verify: Beta team users should NOT see this environment
# When Bob logs in, he only sees alpha-production (his team's env)
# Beta users see nothing here
```

## Step 4: Container-Level Access Control with Labels

```yaml
# Apply team labels to containers for tracking
version: "3.8"

services:
  api:
    image: alpha-corp/api:latest
    labels:
      # Portainer uses these labels for access control display
      - "io.portainer.accesscontrol.teams=$ALPHA_TEAM_ID"
      - "io.portainer.accesscontrol.public=false"
      - "tenant=alpha-corp"
    networks:
      - alpha_net

networks:
  alpha_net:
    driver: bridge
    labels:
      - "tenant=alpha-corp"
```

## Step 5: Registry Access Isolation per Tenant

```bash
# Give each tenant access to their own registry only
# This prevents Alpha Corp from pulling Beta Inc's images

# Add Alpha Corp's private registry
curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/registries" \
  -d "{
    \"Name\": \"Alpha Corp Registry\",
    \"Type\": 1,
    \"URL\": \"registry.alpha-corp.com\",
    \"Authentication\": true,
    \"Username\": \"registry-user\",
    \"Password\": \"registry-pass\"
  }"

# Get the registry ID
ALPHA_REGISTRY_ID=$(curl -s \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  "$PORTAINER_URL/api/registries" | \
  jq -r '.[] | select(.Name == "Alpha Corp Registry") | .Id')

# Grant ONLY Alpha Corp team access to this registry
curl -s -X PUT \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/registries/$ALPHA_REGISTRY_ID/access" \
  -d "{
    \"AuthorizedTeams\": [$ALPHA_TEAM],
    \"AuthorizedUsers\": []
  }"
```

## Step 6: Verify Tenant Isolation

```bash
# Test isolation: authenticate as Alpha team user
ALICE_TOKEN=$(curl -s -X POST \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/auth" \
  -d '{"Username": "alice", "Password": "Alice@Secure123!"}' | \
  jq -r '.jwt')

# Alice should only see Alpha Corp environments
curl -s \
  -H "Authorization: Bearer $ALICE_TOKEN" \
  "$PORTAINER_URL/api/endpoints" | \
  jq '.[].Name'
# Should return ONLY: "alpha-production", "alpha-staging"

# Alice should NOT see Beta's environment
BETA_ENV_VISIBLE=$(curl -s \
  -H "Authorization: Bearer $ALICE_TOKEN" \
  "$PORTAINER_URL/api/endpoints" | \
  jq -r '.[] | select(.Name | contains("beta")) | .Name')

if [ -z "$BETA_ENV_VISIBLE" ]; then
  echo "ISOLATION VERIFIED: Alice cannot see Beta's environments"
else
  echo "WARNING: Isolation breach - Alice can see: $BETA_ENV_VISIBLE"
fi
```

## Conclusion

Portainer's Teams and environment access control model maps naturally to multi-tenant scenarios. Each tenant gets a team, each environment gets access restricted to exactly one team, and team leaders can manage their environment without any visibility into other tenants. The combination of environment-level and container-level access controls provides defense-in-depth — even if a user somehow gains access to an environment, container-level labels and policies provide a second barrier. Validate your tenant isolation periodically by authenticating as a tenant user and confirming they cannot see resources belonging to other tenants.
