# How to Change the Minimum Password Length in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Password Policy, Administration, Configuration

Description: Configure the minimum password length requirement for Portainer user accounts to meet your organization's security policy.

## Introduction

Portainer enforces a minimum password length for all user accounts. By default, this is set to 12 characters. Organizations with stricter security requirements may want to increase this, while development environments might lower it for convenience. This guide shows how to configure the password length policy.

## Finding the Password Policy Settings

Password policy settings are managed through the Portainer web UI under Settings. This is not configurable via CLI flags — it must be set after Portainer is running.

## Step 1: Access Security Settings

1. Log in to Portainer as an administrator
2. Navigate to **Settings** in the left sidebar
3. Select **Authentication**
4. Find the **Security** or **Password rules** section

## Step 2: Configure Minimum Password Length

In the Authentication settings page, look for:

- **Minimum password length**: Set the minimum number of characters (default: 12)

The UI shows a slider or input field where you can set the minimum from 1 to 100 characters.

## Step 3: Enable Additional Password Requirements

Portainer also supports additional password complexity rules:

```
Password rules:
☑ Minimum length: [12]
☑ Require at least one uppercase letter
☑ Require at least one lowercase letter
☑ Require at least one number
☑ Require at least one special character
```

## Configuring via the API

For automation or infrastructure-as-code workflows, use the Portainer API:

```bash
# First, get an authentication token
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Get current settings
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/settings \
  | python3 -m json.tool | grep -i password

# Update minimum password length to 16
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/settings \
  -d '{
    "AuthenticationMethod": 1,
    "ldapsettings": {},
    "oauthsettings": {},
    "openAMTConfiguration": {},
    "SnapshotInterval": "5m",
    "TemplatesURL": "https://raw.githubusercontent.com/portainer/templates/master/templates-2.0.json",
    "EdgeAgentCheckinInterval": 5,
    "EnableEdgeComputeFeatures": false,
    "UserSessionTimeout": "8h",
    "KubeconfigExpiry": "0",
    "KubectlShellImage": "portainer/kubectl-shell",
    "InternalAuthSettings": {
      "RequiredPasswordLength": 16
    }
  }'
```

## Portainer Business Edition Additional Options

Portainer Business Edition provides more granular password policies:

```json
{
  "InternalAuthSettings": {
    "RequiredPasswordLength": 16,
    "PasswordStrengthChecker": {
      "RequireUppercase": true,
      "RequireLowercase": true,
      "RequireDigit": true,
      "RequireSymbol": true
    }
  }
}
```

## Docker Compose with Initial Password Policy

For fresh deployments, you can set the admin password meeting your policy from the start:

```yaml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    command:
      # Set a strong initial admin password (bcrypt hashed)
      - "--admin-password-file=/run/secrets/portainer_admin_password"
    secrets:
      - portainer_admin_password
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

secrets:
  portainer_admin_password:
    file: ./portainer_password.txt

volumes:
  portainer_data:
```

## Best Practices

**Recommended minimums by environment:**

| Environment | Minimum Length | Additional Rules |
|-------------|---------------|-----------------|
| Development | 8 characters | None required |
| Staging | 12 characters | Mixed case |
| Production | 16 characters | Mixed case + digits + symbols |
| High-security | 20 characters | All rules |

## Enforcing Policy for Existing Users

Changing the minimum password length does NOT retroactively invalidate existing passwords. Users with short passwords can still log in. To force all users to update:

1. After changing the policy, notify users to update their passwords
2. Portainer will enforce the new policy when users next change their password
3. Admins can manually reset passwords for users who don't comply

## Conclusion

Portainer's password length policy is a simple but effective control for maintaining account security standards. Configure it via the Settings UI or the API as part of your initial deployment, and align it with your organization's security policy. Remember that policy changes apply to new password creations, not existing passwords.
