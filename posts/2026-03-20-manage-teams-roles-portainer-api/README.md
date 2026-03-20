# How to Manage Teams and Roles via the Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Teams, RBAC, Access Control

Description: Learn how to create teams, manage memberships, and assign environment access using the Portainer REST API.

## Teams API Overview

Teams in Portainer group users together for environment access control. Instead of assigning access user-by-user, you assign it to teams.

| Endpoint | Method | Action |
|----------|--------|--------|
| `/api/teams` | GET | List all teams |
| `/api/teams` | POST | Create a team |
| `/api/teams/{id}` | PUT | Update a team |
| `/api/teams/{id}` | DELETE | Delete a team |
| `/api/team_memberships` | GET | List team memberships |
| `/api/team_memberships` | POST | Add member to team |
| `/api/team_memberships/{id}` | DELETE | Remove member from team |

## Creating a Team

```bash
# Create a new team

curl -X POST "${PORTAINER_URL}/api/teams" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"Name": "frontend-team"}'

# Response
# {"Id": 3, "Name": "frontend-team"}
```

## Listing Teams

```bash
# List all teams
curl -s "${PORTAINER_URL}/api/teams" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | {id: .Id, name: .Name}]'
```

## Adding Users to a Team

```bash
# Add a user to a team
# Role 1 = Team Leader, Role 2 = Team Member
curl -X POST "${PORTAINER_URL}/api/team_memberships" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "TeamID": 3,
    "UserID": 7,
    "Role": 2
  }'

# Add multiple users to a team
for USER_ID in 7 8 9 10; do
  curl -X POST "${PORTAINER_URL}/api/team_memberships" \
    -H "Authorization: Bearer ${API_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{\"TeamID\": 3, \"UserID\": ${USER_ID}, \"Role\": 2}"
done
```

## Listing Team Memberships

```bash
# Get all memberships for a team
curl -s "${PORTAINER_URL}/api/teams/3/memberships" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | {userId: .UserID, role: (if .Role == 1 then "leader" else "member" end)}]'
```

## Removing a User from a Team

```bash
# First, find the membership ID
MEMBERSHIP_ID=$(curl -s "${PORTAINER_URL}/api/team_memberships" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq --argjson teamId 3 --argjson userId 7 \
  '.[] | select(.TeamID == $teamId and .UserID == $userId) | .Id')

# Delete the membership
curl -X DELETE "${PORTAINER_URL}/api/team_memberships/${MEMBERSHIP_ID}" \
  -H "Authorization: Bearer ${API_TOKEN}"
```

## Granting Team Access to an Environment

```bash
# Grant a team access to a specific environment
curl -X PUT "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/access" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"TeamAccessPolicies\": {
      \"3\": {\"RoleId\": 2}
    }
  }"
```

## Full Onboarding Automation

```bash
#!/bin/bash
# Onboard a new developer: create user, add to team, grant environment access

API_TOKEN="${PORTAINER_API_TOKEN}"
PORTAINER_URL="https://portainer.mycompany.com"

USERNAME="$1"
TEAM_NAME="$2"
ENDPOINT_ID="$3"

# Create user
USER_ID=$(curl -s -X POST "${PORTAINER_URL}/api/users" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"Username\":\"${USERNAME}\",\"Password\":\"$(openssl rand -base64 12)\",\"Role\":2}" \
  | jq '.Id')

# Find team ID
TEAM_ID=$(curl -s "${PORTAINER_URL}/api/teams" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq --arg name "$TEAM_NAME" '.[] | select(.Name == $name) | .Id')

# Add to team
curl -s -X POST "${PORTAINER_URL}/api/team_memberships" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"TeamID\":${TEAM_ID},\"UserID\":${USER_ID},\"Role\":2}"

echo "Onboarded ${USERNAME} (ID: ${USER_ID}) to team ${TEAM_NAME}"
```

## Conclusion

The Portainer teams and roles API enables automated user provisioning that scales with your team. Connect it to your HR system or identity provider to automatically grant and revoke access as people join and leave the organization.
