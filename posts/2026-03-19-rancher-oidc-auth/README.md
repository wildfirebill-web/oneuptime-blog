# How to Configure OIDC Authentication in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Authentication, OIDC, SSO, RBAC

Description: Learn how to set up OpenID Connect (OIDC) authentication in Rancher for modern token-based single sign-on.

OpenID Connect (OIDC) is a modern authentication protocol built on top of OAuth 2.0. Rancher supports OIDC authentication with various identity providers including Keycloak, Auth0, Dex, and any OIDC-compliant provider. This guide covers configuring OIDC authentication in Rancher.

## Prerequisites

- Rancher v2.6 or later with admin access
- An OIDC-compatible identity provider
- The Rancher server accessible via HTTPS
- Admin access to your OIDC provider to create client applications

## Understanding OIDC vs SAML

While SAML uses XML-based assertions, OIDC uses JSON Web Tokens (JWT). OIDC is generally simpler to configure and better suited for modern applications. Rancher supports both, but OIDC is the recommended approach for new integrations with compatible providers.

## Step 1: Register Rancher in Your OIDC Provider

Create a client application in your OIDC provider. The following example uses Keycloak:

1. Log in to the Keycloak Admin Console.
2. Navigate to **Clients** and click **Create client**.

```
Client Type: OpenID Connect
Client ID: rancher
Client Authentication: ON (confidential)
```

3. Configure the redirect URI:

```
Valid Redirect URIs: https://rancher.example.com/verify-auth
Web Origins: https://rancher.example.com
```

4. Note the client credentials:

```
Client ID: rancher
Client Secret: (from the Credentials tab)
```

For Auth0:

1. Create a new Application (Regular Web Application).
2. Set the callback URL to `https://rancher.example.com/verify-auth`.
3. Note the Client ID, Client Secret, and Domain.

## Step 2: Discover OIDC Endpoints

Find your provider's OIDC discovery endpoint:

```bash
# Keycloak
curl -s "https://keycloak.example.com/realms/your-realm/.well-known/openid-configuration" | jq

# Auth0
curl -s "https://your-domain.auth0.com/.well-known/openid-configuration" | jq

# Dex
curl -s "https://dex.example.com/.well-known/openid-configuration" | jq
```

The discovery document provides all required endpoints:

```json
{
  "issuer": "https://keycloak.example.com/realms/your-realm",
  "authorization_endpoint": "https://keycloak.example.com/realms/your-realm/protocol/openid-connect/auth",
  "token_endpoint": "https://keycloak.example.com/realms/your-realm/protocol/openid-connect/token",
  "userinfo_endpoint": "https://keycloak.example.com/realms/your-realm/protocol/openid-connect/userinfo",
  "jwks_uri": "https://keycloak.example.com/realms/your-realm/protocol/openid-connect/certs",
  "end_session_endpoint": "https://keycloak.example.com/realms/your-realm/protocol/openid-connect/logout"
}
```

## Step 3: Configure Group Claims

Ensure your OIDC provider sends group information in the ID token or userinfo response.

For Keycloak:

1. In the Rancher client, go to **Client scopes**.
2. Add a mapper:

```
Mapper Type: Group Membership
Name: groups
Token Claim Name: groups
Full group path: OFF
Add to ID token: ON
Add to access token: ON
Add to userinfo: ON
```

For Auth0, create a custom rule or action:

```javascript
// Auth0 Action - Add groups to token
exports.onExecutePostLogin = async (event, api) => {
  const namespace = 'https://rancher.example.com';
  if (event.authorization) {
    api.idToken.setCustomClaim(`${namespace}/groups`, event.authorization.roles);
    api.accessToken.setCustomClaim(`${namespace}/groups`, event.authorization.roles);
  }
};
```

## Step 4: Configure OIDC in Rancher

Set up the OIDC authentication provider:

1. Log in to Rancher as an administrator.
2. Navigate to **Users & Authentication** then **Auth Provider**.
3. Select **Keycloak (OIDC)** or the generic OIDC option if available.

Enter the configuration:

```
Client ID: rancher
Client Secret: <your-client-secret>
Issuer URL: https://keycloak.example.com/realms/your-realm
Auth Endpoint: https://keycloak.example.com/realms/your-realm/protocol/openid-connect/auth
Token Endpoint: https://keycloak.example.com/realms/your-realm/protocol/openid-connect/token
User Info Endpoint: https://keycloak.example.com/realms/your-realm/protocol/openid-connect/userinfo
JWKS URL: https://keycloak.example.com/realms/your-realm/protocol/openid-connect/certs
Rancher URL: https://rancher.example.com
```

## Step 5: Configure Claim Mappings

Map OIDC claims to Rancher user attributes:

```
UID Claim: sub
Display Name Claim: name
User Name Claim: preferred_username
Groups Claim: groups
Email Claim: email
```

For Auth0 with custom claims:

```
Groups Claim: https://rancher.example.com/groups
```

## Step 6: Configure Scopes

Set the OIDC scopes that Rancher requests:

```
Scopes: openid profile email groups
```

For providers that use different scope names:

```
# Keycloak (default scopes)
Scopes: openid profile email

# Auth0
Scopes: openid profile email

# Dex
Scopes: openid profile email groups offline_access
```

## Step 7: Test the OIDC Configuration

Test authentication before enabling:

1. Click the **Test** button.
2. You will be redirected to your OIDC provider's login page.
3. Authenticate with your credentials.
4. Verify that you are redirected back to Rancher with correct user information.

Check the returned claims:

```bash
# Decode a JWT token to verify claims (for debugging)
echo "$TOKEN" | cut -d '.' -f 2 | base64 -d 2>/dev/null | jq

# Check Rancher logs
kubectl logs -l app=rancher -n cattle-system --tail=200 | grep -i "oidc\|auth\|keycloak"
```

## Step 8: Enable OIDC Authentication

After successful testing:

1. Click **Enable** to activate OIDC authentication.
2. Confirm the action.

The login page will show the OIDC login button.

## Step 9: Map OIDC Groups to Rancher Roles

Assign roles based on OIDC groups:

1. Navigate to **Users & Authentication** then **Groups**.
2. Search for OIDC groups.
3. Assign roles.

```
OIDC Group: platform-admins -> Administrator
OIDC Group: devops -> Standard User
OIDC Group: developers -> Standard User
OIDC Group: viewers -> User-Base
```

For cluster-level access:

1. Navigate to a cluster and go to **Cluster Members**.
2. Add the OIDC group with the desired cluster role.

## Step 10: Advanced OIDC Configuration

### Token Refresh

Configure token refresh behavior:

```bash
# Set token TTL
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "57600"}' \
  "https://rancher.example.com/v3/settings/auth-token-max-ttl-minutes"
```

### Logout Integration

Configure end-session for proper logout:

Ensure the OIDC provider's end-session endpoint is configured so that logging out of Rancher also ends the OIDC session.

### Custom CA Certificates

If your OIDC provider uses a private CA:

1. Add the CA certificate to Rancher.
2. In the OIDC configuration, paste the CA certificate.

Or add it to the Rancher deployment:

```bash
kubectl create secret generic tls-ca-additional \
  -n cattle-system \
  --from-file=ca-additional.pem=oidc-ca.pem
```

## Troubleshooting

Common OIDC issues:

| Issue | Cause | Solution |
|-------|-------|----------|
| Invalid redirect URI | Redirect mismatch | Ensure the redirect URI is exactly `https://rancher.example.com/verify-auth` |
| Token validation failed | Clock skew or wrong JWKS | Sync NTP and verify the JWKS URL |
| Groups not returned | Missing scope or mapper | Add the groups scope and verify the claim mapper |
| Invalid client credentials | Wrong client secret | Regenerate and update the client secret |
| SSL handshake error | Untrusted CA | Add the CA certificate to Rancher |

## Best Practices

- **Use discovery endpoint**: Let Rancher discover endpoints automatically from the `.well-known/openid-configuration` URL when possible.
- **Configure proper scopes**: Only request the scopes you need to minimize the data exchanged.
- **Map groups to roles**: Use OIDC groups for role assignments instead of individual user mappings.
- **Monitor token expiration**: Configure appropriate token lifetimes in both the OIDC provider and Rancher.
- **Test logout flow**: Verify that logging out of Rancher properly terminates the OIDC session.

## Conclusion

OIDC authentication in Rancher provides a modern, standards-based approach to single sign-on. With support for JWT tokens, automatic key rotation via JWKS, and broad provider compatibility, OIDC is an excellent choice for integrating Rancher with your identity infrastructure. Configure your provider's group claims carefully to enable seamless role-based access control in your Kubernetes management platform.
