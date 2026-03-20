# How to Hide Internal Authentication When Using OAuth in Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, OAuth, SSO, Authentication, Security, UX

Description: Hide the internal username/password login form in Portainer when OAuth is the primary authentication method, while keeping emergency admin access available.

## Introduction

When OAuth is your primary authentication method, showing both the OAuth button and the internal login form can confuse users and suggest an alternative (less secure for regular users) login path. Portainer's `HideInternalAuth` option removes the internal login form from the login page, directing all users through OAuth. This guide explains how to configure this and maintain emergency admin access.

## Understanding HideInternalAuth

When `HideInternalAuth` is enabled:
- The username/password login form is hidden from the login page
- Users only see the OAuth/SSO login button
- The internal admin account still exists and is accessible via a URL parameter
- Emergency access is preserved

This does NOT:
- Delete or disable internal user accounts
- Prevent API access with JWT tokens from internal accounts
- Remove admin capabilities from internal accounts

## Enabling HideInternalAuth via UI

1. Go to Settings → Authentication → OAuth
2. Configure OAuth settings
3. Enable **Hide internal authentication prompt**
4. Save settings

## Enabling via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/settings \
  -d '{
    "AuthenticationMethod": 3,
    "oauthsettings": {
      "ClientID": "your-client-id",
      "ClientSecret": "your-client-secret",
      "AuthorizationURI": "https://idp.example.com/oauth/authorize",
      "AccessTokenURI": "https://idp.example.com/oauth/token",
      "ResourceURI": "https://idp.example.com/oauth/userinfo",
      "RedirectURI": "https://portainer.example.com/",
      "UserIdentifier": "email",
      "Scopes": "openid profile email",
      "OAuthAutoCreateUsers": true,
      "SSO": true,
      "HideInternalAuth": true
    }
  }'
```

## Accessing Internal Login When Hidden

Even with `HideInternalAuth` enabled, you can access the internal login form by appending a query parameter:

```text
https://portainer.example.com/?skipSSO=true
```

This displays the internal username/password form regardless of the `HideInternalAuth` setting.

**Document this URL** in your runbooks and share it only with admins who need emergency access.

## Emergency Admin Account Best Practices

Since `HideInternalAuth` makes the internal admin less visible, ensure you:

1. **Have a strong admin password** documented in a password manager or secrets vault
2. **Test emergency access regularly** - log in via `?skipSSO=true` quarterly
3. **Have at least two admin accounts** in case one account is compromised or password is forgotten
4. **Don't rely on the IdP admin** - your IdP being unavailable is exactly the scenario where you need internal admin access

```bash
# Create a backup admin account before enabling HideInternalAuth

curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/users \
  -d '{
    "username": "emergency-admin",
    "password": "VeryStrongEmergencyP@ssword123!",
    "role": 1
  }'

# Test that emergency access works
curl -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"emergency-admin","password":"VeryStrongEmergencyP@ssword123!"}' \
  | python3 -m json.tool
```

## Combining with SSO

For the cleanest experience, combine both SSO and HideInternalAuth:

```json
{
  "SSO": true,
  "HideInternalAuth": true
}
```

With both enabled:
1. User visits Portainer
2. If no active OAuth session: redirected to IdP immediately
3. User authenticates with IdP
4. Redirected back and logged in
5. The Portainer login page is never shown

## Reverting HideInternalAuth

If the IdP is unavailable and you need to re-enable the internal login form for all users:

```bash
# Use the emergency admin token (from ?skipSSO=true login)
curl -X PUT \
  -H "Authorization: Bearer $EMERGENCY_TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/settings \
  -d '{
    "oauthsettings": {
      "HideInternalAuth": false
    }
  }'
```

## Conclusion

`HideInternalAuth` provides a cleaner user experience when OAuth is the primary login method, guiding all users through the standardized SSO flow. The `?skipSSO=true` escape hatch ensures you're never completely locked out if the IdP is unavailable. Always test emergency access before enabling this feature in production and document the emergency URL in your incident response procedures.
