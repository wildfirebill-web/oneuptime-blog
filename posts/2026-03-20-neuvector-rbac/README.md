# How to Configure NeuVector RBAC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, RBAC, Access Control, Kubernetes, Security

Description: Configure NeuVector's role-based access control to assign fine-grained permissions to users and teams for managing container security policies.

## Introduction

NeuVector's Role-Based Access Control (RBAC) system provides fine-grained access management for security operations. You can assign global roles, namespace-specific roles, and custom roles to users and groups, ensuring that each team member has exactly the access they need - no more, no less.

## NeuVector Built-in Roles

| Role | Description |
|---|---|
| Admin | Full access to all features |
| Reader | Read-only access to all features |
| CI Ops | Access to vulnerability scan API for CI/CD integration |
| Runtime Security | Manage policies, view events, no admin settings |
| Compliance | View and run compliance scans |

## Prerequisites

- NeuVector installed and running
- Admin access to NeuVector Manager
- Understanding of your team's access requirements

## Step 1: Create a User

```bash
# Create a new user with Reader role

curl -sk -X POST \
  "https://neuvector-manager:8443/v1/user" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "username": "security-engineer",
      "password": "SecurePass123!",
      "fullname": "Jane Smith",
      "email": "jane.smith@company.com",
      "role": "reader",
      "timeout": 900,
      "locale": "en",
      "role_domains": {}
    }
  }'
```

In the UI:
1. Go to **Settings** > **Users & Roles**
2. Click **Add User**
3. Fill in username, password, full name, email
4. Select the global role

## Step 2: Assign Global Roles

```bash
# Assign admin role to a user
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/user/security-engineer" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "role": "admin"
    }
  }'

# Assign reader role
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/user/dev-team-lead" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "role": "reader"
    }
  }'
```

## Step 3: Configure Namespace-Scoped Roles

Assign users different roles for specific namespaces:

```bash
# Give user admin access only in the "staging" namespace
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/user/staging-admin" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "role": "",
      "role_domains": {
        "admin": ["staging"],
        "reader": ["production", "default"]
      }
    }
  }'
```

This grants:
- Admin access in the `staging` namespace
- Reader access in `production` and `default`
- No access to other namespaces

## Step 4: Configure CI/CD Service Account

Create a dedicated account for CI/CD pipeline integration:

```bash
# Create CI/CD service account with ciops role
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/user" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "username": "ci-scanner",
      "password": "CIScannerSecurePass456!",
      "fullname": "CI/CD Scanner",
      "email": "security-automation@company.com",
      "role": "ciops",
      "timeout": 300
    }
  }'
```

## Step 5: List and Manage Users

```bash
# List all users
curl -sk \
  "https://neuvector-manager:8443/v1/user" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.users[] | {
    username: .username,
    fullname: .fullname,
    role: .role,
    email: .email,
    blocked: .blocked_for_failed_login
  }'

# Disable a user (without deleting)
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/user/old-employee" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "blocked_for_failed_login": true
    }
  }'

# Delete a user
curl -sk -X DELETE \
  "https://neuvector-manager:8443/v1/user/old-employee" \
  -H "X-Auth-Token: ${TOKEN}"
```

## Step 6: Configure Password Policy

Set password requirements for all users:

```bash
# Configure password policy
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/system/config" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "auth_order": ["local"],
      "auth_by_platform": false,
      "rancher_ep": "",
      "password_policy": {
        "enable_password_policy": true,
        "min_len": 16,
        "require_uppercase": true,
        "require_lowercase": true,
        "require_digit": true,
        "require_special_character": true,
        "password_expire_after_days": 90,
        "password_keep_history": 5
      }
    }
  }'
```

## Step 7: Configure Session Settings

```bash
# Set idle timeout and maximum login attempts
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/system/config" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "webhook_status": false,
      "auth_by_platform": false,
      "password_policy": {
        "enable_password_policy": true,
        "block_after_failed_login_count": 5,
        "block_minutes": 30
      }
    }
  }'
```

## Step 8: Use Kubernetes RBAC with NeuVector

Integrate NeuVector RBAC with Kubernetes service accounts:

```yaml
# neuvector-rbac-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: neuvector-reader-role
rules:
  - apiGroups: ["neuvector.com"]
    resources: ["nvsecurityrules", "nvclusterSecurityrules"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dev-team-neuvector-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: neuvector-reader-role
subjects:
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
```

## Conclusion

NeuVector's RBAC system provides the flexibility to implement least-privilege access across your security operations team. By assigning namespace-scoped roles, you give development teams visibility into their own namespaces without exposing sensitive production security policies. Combined with LDAP/AD integration and strong password policies, NeuVector RBAC supports enterprise identity management requirements.
