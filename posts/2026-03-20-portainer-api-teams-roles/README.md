# How to Manage Teams and Roles via the Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Teams, RBAC, Automation

Description: Learn how to create and manage Portainer teams, assign users to teams, and configure role-based access to environments via the REST API for automated RBAC management.

## Introduction

Portainer's team-based RBAC system allows you to group users and grant them access to environments at different permission levels. The Portainer API provides full control over team and role management, enabling automated provisioning as new projects or teams are onboarded.

## Prerequisites

- Portainer CE or BE with admin access
- Valid admin JWT token or API access token
- Users already created in Portainer

## Understanding Portainer RBAC

```
Teams → Assigned to Environments with a Role
                ↓
Role values:
  1 = Environment Administrator
  2 = Standard User (can deploy)
  3 = Read-only User (view only)
```

## Step 1: List All Teams

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"

# List all teams
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/teams" | jq '.[] | {id: .Id, name: .Name}'
```

## Step 2: Create a Team

```bash
# Create a new team
TEAM=$(curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/teams" \
  -d '{"name": "backend-engineers"}')

TEAM_ID=$(echo $TEAM | jq -r '.Id')
echo "Team created with ID: $TEAM_ID"

# Create multiple teams
for TEAM_NAME in "frontend" "backend" "devops" "qa"; do
  RESULT=$(curl -s -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "${PORTAINER_URL}/api/teams" \
    -d "{\"name\": \"${TEAM_NAME}\"}")
  echo "Created team '${TEAM_NAME}': ID=$(echo $RESULT | jq -r '.Id')"
done
```

## Step 3: Add Users to a Team

```bash
TEAM_ID=2
USER_ID=5

# Add a user to a team as a regular member (role 2)
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/teams/${TEAM_ID}/memberships" \
  -d "{
    \"userId\": $USER_ID,
    \"role\": 2
  }" | jq .

# Add a user as team leader (role 1)
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/teams/${TEAM_ID}/memberships" \
  -d "{
    \"userId\": $USER_ID,
    \"role\": 1
  }" | jq .
```

## Step 4: List Team Members

```bash
TEAM_ID=2

# List all members of a team
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/teams/${TEAM_ID}/memberships" | jq .

# Get membership details with user info
MEMBERSHIPS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/teams/${TEAM_ID}/memberships")

echo "Team $TEAM_ID members:"
echo $MEMBERSHIPS | jq -r '.[] | "  User ID: \(.UserID), Role: \(if .Role == 1 then "Leader" else "Member" end)"'
```

## Step 5: Remove a User from a Team

```bash
MEMBERSHIP_ID=10

# Remove a team membership
curl -s -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/team_memberships/${MEMBERSHIP_ID}"

echo "Membership removed."
```

## Step 6: Grant Team Access to an Environment

After creating a team, grant it access to specific environments:

```bash
ENDPOINT_ID=1
TEAM_ID=2

# Grant team access to an environment with role 2 (Standard User)
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/teamaccesspolicies" \
  -d "{
    \"$TEAM_ID\": {\"RoleId\": 2}
  }" | jq .
```

## Step 7: Grant User Direct Access to an Environment

```bash
USER_ID=5
ENDPOINT_ID=1

# Grant individual user access to an environment
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/useraccesspolicies" \
  -d "{
    \"$USER_ID\": {\"RoleId\": 2}
  }" | jq .
```

## Step 8: Full RBAC Setup Script

```bash
#!/bin/bash
# setup-rbac.sh — Configure teams and access policies

PORTAINER_URL="https://portainer.example.com"
TOKEN=$(curl -s -X POST "${PORTAINER_URL}/api/auth" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' | jq -r '.jwt')

# Configuration
PROD_ENDPOINT_ID=1
STAGING_ENDPOINT_ID=2

echo "=== Setting up RBAC ==="

# Create teams
for TEAM_NAME in "devops" "backend" "frontend"; do
  TEAM=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "${PORTAINER_URL}/api/teams" \
    -d "{\"name\": \"$TEAM_NAME\"}")
  TEAM_ID=$(echo $TEAM | jq -r '.Id')
  echo "Created team '$TEAM_NAME' (ID: $TEAM_ID)"

  # Grant staging access to all teams (role 2 = standard)
  curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "${PORTAINER_URL}/api/endpoints/${STAGING_ENDPOINT_ID}/teamaccesspolicies" \
    -d "{\"$TEAM_ID\": {\"RoleId\": 2}}" > /dev/null
  echo "  Granted staging access to '$TEAM_NAME'"

  # Grant production access only to devops team (role 1 = admin)
  if [ "$TEAM_NAME" = "devops" ]; then
    curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      "${PORTAINER_URL}/api/endpoints/${PROD_ENDPOINT_ID}/teamaccesspolicies" \
      -d "{\"$TEAM_ID\": {\"RoleId\": 1}}" > /dev/null
    echo "  Granted production ADMIN access to 'devops'"
  fi
done

echo "=== RBAC setup complete ==="
```

## Step 9: Delete a Team

```bash
TEAM_ID=2

# Delete a team (removes team and all its memberships)
curl -s -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/teams/${TEAM_ID}"

echo "Team $TEAM_ID deleted."
```

## Conclusion

Managing teams and roles via the Portainer API enables a consistent, automated approach to access control. Create teams aligned with your organizational structure, add users programmatically during onboarding, and grant environment access at appropriate privilege levels. This approach scales well as your team and infrastructure grow, and integrates cleanly with HR or directory systems through custom provisioning scripts.
