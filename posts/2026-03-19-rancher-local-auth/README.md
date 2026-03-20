# How to Set Up Local Authentication in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Authentication, RBAC

Description: A guide to configuring and managing local authentication in Rancher for user management without an external identity provider.

Local authentication is Rancher's built-in user management system. It allows you to create and manage user accounts directly within Rancher without relying on an external identity provider. This guide covers setting up local authentication, managing users, configuring password policies, and securing local accounts.

## Prerequisites

- Rancher v2.6 or later
- Admin access to Rancher
- Understanding of Rancher's role-based access control

## Understanding Local Authentication

Local authentication is enabled by default when you first install Rancher. The initial admin user created during setup uses local authentication. Even when an external auth provider is configured, local authentication remains available as a fallback, ensuring administrators can always access Rancher.

## Step 1: Access User Management

Navigate to the user management area:

1. Log in to Rancher as an administrator.
2. Click the hamburger menu.
3. Select **Users & Authentication**.
4. Click **Users** to see the user list.

## Step 2: Create Local Users

Add new local users:

1. Click **Create**.
2. Fill in the user details:

```plaintext
Username: jdoe
Display Name: John Doe
Description: Backend developer

Password: <strong-password>
Confirm Password: <strong-password>
```

3. Assign a global role:

```plaintext
☑ Standard User
☐ Administrator
☐ User-Base
☐ Custom
```

4. Click **Create**.

Create multiple users via the Rancher API:

```bash
# Create a local user via API

curl -s -k \
  -X POST \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "jdoe",
    "password": "SecureP@ssw0rd!",
    "name": "John Doe",
    "description": "Backend developer",
    "mustChangePassword": true
  }' \
  "https://rancher.example.com/v3/users"
```

## Step 3: Configure Password Requirements

Set password complexity requirements:

1. Navigate to **Global Settings**.
2. Find the password-related settings.

```bash
# Set minimum password length
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "12"}' \
  "https://rancher.example.com/v3/settings/password-min-length"
```

## Step 4: Force Password Changes

Require users to change their password on first login:

When creating users via the API, set `mustChangePassword` to `true`:

```bash
curl -s -k \
  -X POST \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "newuser",
    "password": "TemporaryP@ss1",
    "name": "New User",
    "mustChangePassword": true
  }' \
  "https://rancher.example.com/v3/users"
```

To force an existing user to change their password:

```bash
# Update user to require password change
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mustChangePassword": true}' \
  "https://rancher.example.com/v3/users/<user-id>"
```

## Step 5: Assign Cluster and Project Roles

Grant local users access to specific clusters and projects:

### Cluster Access

1. Navigate to the target cluster.
2. Go to **Cluster Members**.
3. Click **Add**.
4. Search for the local user.
5. Select the cluster role.

```plaintext
User: jdoe
Cluster Role: Cluster Member

User: sysadmin
Cluster Role: Cluster Owner
```

### Project Access

1. Navigate to the cluster and then the target project.
2. Go to **Members**.
3. Click **Add**.
4. Search for the local user.
5. Select the project role.

```plaintext
User: jdoe
Project: backend-services
Project Role: Project Member

User: jdoe
Project: frontend-services
Project Role: Read Only
```

## Step 6: Manage API Keys

Create API keys for local users:

1. Click the user avatar in the top-right corner.
2. Select **Account & API Keys**.
3. Click **Create API Key**.

```plaintext
Description: CI/CD Pipeline Access
Scope: No Scope (access all clusters)
Expires: In 90 days
```

Via the API:

```bash
# Create an API key
curl -s -k \
  -X POST \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "CI/CD Pipeline Access",
    "ttl": 7776000000
  }' \
  "https://rancher.example.com/v3/tokens"
```

The response includes the access key and secret key:

```json
{
  "token": "token-xxxxx:xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
}
```

## Step 7: Disable and Delete Users

Manage user lifecycle:

### Disable a User

1. Navigate to **Users & Authentication** then **Users**.
2. Find the user.
3. Click the three-dot menu and select **Deactivate**.

```bash
# Disable via API
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"enabled": false}' \
  "https://rancher.example.com/v3/users/<user-id>"
```

### Delete a User

1. Find the user in the user list.
2. Click the three-dot menu and select **Delete**.
3. Confirm the deletion.

```bash
# Delete via API
curl -s -k \
  -X DELETE \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://rancher.example.com/v3/users/<user-id>"
```

## Step 8: Reset Admin Password

If you lose the admin password, reset it using kubectl:

```bash
# Reset the admin password
kubectl -n cattle-system exec $(kubectl -n cattle-system get pods \
  -l app=rancher --no-headers | head -1 | awk '{ print $1 }') \
  -- reset-password

# The command outputs a new temporary password
# New password for default admin user (user-xxxxx): <new-password>
```

Alternatively, set a specific password:

```bash
# Get the admin user ID
ADMIN_ID=$(kubectl -n cattle-system exec $(kubectl -n cattle-system get pods \
  -l app=rancher --no-headers | head -1 | awk '{ print $1 }') \
  -- loglevel --set info 2>&1 | grep -o 'user-[a-z0-9]*' | head -1)

# Use the Rancher CLI to reset
kubectl -n cattle-system exec $(kubectl -n cattle-system get pods \
  -l app=rancher --no-headers | head -1 | awk '{ print $1 }') \
  -- reset-password
```

## Step 9: Configure Session Settings

Manage session duration and behavior:

```bash
# Set session token TTL (in minutes, 0 for no expiry)
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "960"}' \
  "https://rancher.example.com/v3/settings/auth-token-max-ttl-minutes"

# Set the maximum number of concurrent sessions
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "0"}' \
  "https://rancher.example.com/v3/settings/kubeconfig-default-token-ttl-minutes"
```

## Step 10: Audit Local User Activity

Monitor user activity through Rancher logs:

```bash
# View authentication events
kubectl logs -l app=rancher -n cattle-system --tail=500 | \
  grep -i "login\|auth\|user" | tail -20

# List all users and their status
curl -s -k \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://rancher.example.com/v3/users" | \
  jq '.data[] | {
    username: .username,
    name: .name,
    enabled: .enabled,
    principalIds: .principalIds,
    created: .created
  }'

# List all API tokens
curl -s -k \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://rancher.example.com/v3/tokens" | \
  jq '.data[] | {
    name: .name,
    userId: .userId,
    description: .description,
    expired: .expired,
    expiresAt: .expiresAt
  }'
```

## Using Local Auth with External Providers

Local authentication can coexist with external providers:

- The admin user always has local auth access as a fallback.
- Users can have both local and external identities.
- If the external auth provider goes down, local accounts still work.

To configure this:

1. Set up your external auth provider (LDAP, SAML, OIDC, etc.).
2. Keep the local admin account active.
3. Create a few emergency local accounts for break-glass scenarios.

## Best Practices

- **Use external auth for most users**: Local authentication is best used as a fallback, not the primary authentication method for large organizations.
- **Keep admin local accounts**: Always maintain at least one local admin account for emergency access.
- **Enforce strong passwords**: Set minimum password length and require complexity.
- **Rotate API keys**: Set expiration dates on API keys and rotate them regularly.
- **Audit regularly**: Review the user list and remove inactive accounts promptly.
- **Limit admin accounts**: Keep the number of local admin accounts to the minimum necessary.

## Conclusion

Local authentication in Rancher provides a straightforward way to manage user access without external dependencies. While it is ideal for small teams and development environments, larger organizations should consider integrating an external identity provider and using local auth as a fallback. By following the security practices outlined in this guide, you can maintain a secure and well-managed local authentication setup for your Rancher environment.
