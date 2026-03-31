# How to Set Up Multi-Environment Policies in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Environment, Policies, Governance, Compliance

Description: Implement consistent policies across multiple Portainer environments using groups, tags, and standardized access control configurations.

## Introduction

In organizations with many environments, maintaining consistent policies (who can access what, with which role) becomes challenging. Portainer's environment groups and tags provide tools for applying policies consistently at scale. This guide covers strategies for multi-environment policy management.

## Strategy 1: Group-Based Policy Inheritance

Create environment groups and assign access policies at the group level:

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Create environment groups

curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoint_groups \
  -d '{"name":"Production","description":"All production environments","TeamAccessPolicies":{"1":{"RoleId":2},"2":{"RoleId":1}}}'

curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoint_groups \
  -d '{"name":"Development","description":"All dev environments","TeamAccessPolicies":{"1":{"RoleId":2},"2":{"RoleId":2},"3":{"RoleId":2}}}'
```

## Strategy 2: Scripted Policy Application

Apply consistent policies across all environments via script:

```bash
#!/bin/bash
# apply-environment-policies.sh

TOKEN="your-admin-token"
PORTAINER_URL="https://portainer.example.com"

# Get all environments
ENDPOINTS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints" \
  | python3 -c "import sys,json; [print(e['Id']) for e in json.load(sys.stdin)]")

# Standard policy: DevOps (team 1) = Standard User, Support (team 2) = Helpdesk
STANDARD_POLICY='{"1":{"RoleId":2},"2":{"RoleId":1}}'

for endpoint_id in $ENDPOINTS; do
  echo "Applying standard policy to endpoint $endpoint_id"
  curl -s -X PUT \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "${PORTAINER_URL}/api/endpoints/${endpoint_id}/teamaccesspolicies" \
    -d "$STANDARD_POLICY"
done

echo "Done applying policies to all environments"
```

## Strategy 3: Environment Template Automation

When adding new environments, use a script that applies the standard policy automatically:

```bash
#!/bin/bash
# add-environment-with-policy.sh

add_environment_with_policy() {
  local name=$1
  local agent_url=$2
  local group_id=$3

  # Add the environment
  ENDPOINT_ID=$(curl -s -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "${PORTAINER_URL}/api/endpoints" \
    -d "{\"name\":\"${name}\",\"endpointCreationType\":2,\"URL\":\"${agent_url}\"}" \
    | python3 -c "import sys,json; print(json.load(sys.stdin)['Id'])")

  # Add to group (inherits group policies)
  curl -s -X POST \
    -H "Authorization: Bearer $TOKEN" \
    "${PORTAINER_URL}/api/endpoint_groups/${group_id}/endpoints/${ENDPOINT_ID}"

  echo "Added environment $name (ID: $ENDPOINT_ID) to group $group_id"
}

# Usage
add_environment_with_policy "US-West Production" "tcp://us-west-prod:9001" 1
add_environment_with_policy "US-West Staging" "tcp://us-west-staging:9001" 2
```

## Conclusion

Multi-environment policy management in Portainer works best through groups that carry access policies. New environments added to a group inherit the group's policies automatically. For custom policies per-environment, maintain a policy configuration file and apply it via script to ensure consistency and auditability.
