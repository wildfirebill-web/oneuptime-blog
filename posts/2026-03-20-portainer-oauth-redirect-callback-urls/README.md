# How to Configure OAuth Redirect and Callback URLs in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, OAuth, Redirect URI, Callback URL, Configuration

Description: Understand and correctly configure OAuth redirect URIs in Portainer and your identity provider to prevent redirect_uri_mismatch errors.

## Introduction

The redirect URI (callback URL) is one of the most common sources of OAuth configuration errors. It must match exactly between your identity provider registration and Portainer's settings. A single difference in trailing slashes, scheme, or path causes authentication to fail entirely. This guide explains redirect URIs in depth.

## What Is a Redirect URI?

After a user authenticates with the identity provider, the IdP redirects them back to your application using the redirect URI. This URI is a security measure — the IdP only redirects to pre-approved URIs. If the URI Portainer requests doesn't match what's registered with the IdP, authentication fails with a "redirect_uri_mismatch" error.

## Portainer's Redirect URI

Portainer's redirect URI is always:

```
https://your-portainer-domain.com/
```

Key points:
- Uses the same scheme and port as your Portainer URL
- Points to the root of Portainer (no path)
- **Must include a trailing slash** (this is the most common mistake)

## What Must Match Exactly

| Component | Example | Must Match? |
|-----------|---------|------------|
| Scheme | `https://` | Yes |
| Hostname | `portainer.example.com` | Yes |
| Port | `:443` (implicit for HTTPS) | Yes |
| Path | `/` | Yes |
| Trailing slash | `https://portainer.example.com/` | Yes |

Even a missing trailing slash causes failure with most IdPs.

## Configuring in Portainer

In Settings → Authentication → OAuth:

```
Redirect URL: https://portainer.example.com/
```

Or via API:

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
      "RedirectURI": "https://portainer.example.com/",
      ...other settings...
    }
  }'
```

## Registering in Each IdP

### Azure AD

1. App Registration → **Authentication** → **Redirect URIs**
2. Add: `https://portainer.example.com/`
3. Type: **Web**

### Google OAuth

1. In your OAuth client credentials → **Authorized redirect URIs**
2. Add: `https://portainer.example.com/`

### GitHub

1. OAuth App settings → **Authorization callback URL**
2. Value: `https://portainer.example.com/`

### Keycloak

1. Client settings → **Valid redirect URIs**
2. Add: `https://portainer.example.com/`
3. Also add to **Web origins**: `https://portainer.example.com`

### Authentik

1. Provider settings → **Redirect URIs/Origins**
2. Add: `https://portainer.example.com/`

## Common Mistakes and Fixes

### Missing Trailing Slash

```
Wrong:   https://portainer.example.com
Correct: https://portainer.example.com/
```

### Wrong Scheme

```
Wrong:   http://portainer.example.com/   (if running on HTTPS)
Correct: https://portainer.example.com/
```

### Non-Standard Port

```
Wrong:   https://portainer.example.com/   (if running on port 8443)
Correct: https://portainer.example.com:8443/
```

### URL Behind Reverse Proxy

If Portainer is behind a proxy on a different port:
```
Wrong (internal port): https://portainer.example.com:9443/
Correct (public URL):  https://portainer.example.com/
```

Always use the URL that users see in their browser, not internal ports.

## Diagnosing Redirect URI Errors

When you get a "redirect_uri_mismatch" error:

1. Note the exact error message — most IdPs show the "requested" and "expected" URIs
2. Compare them character by character
3. Update whichever side is different

```bash
# Check what Portainer is configured to send
curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/settings \
  | python3 -c "import sys,json; s=json.load(sys.stdin); print(s.get('oauthsettings',{}).get('RedirectURI','not set'))"
```

## Subpath Deployment

If Portainer runs at a subpath (`https://example.com/portainer/`), the redirect URI must be the full subpath:

```
Redirect URL: https://example.com/portainer/
```

Also ensure `--base-url=/portainer` is set in Portainer's startup command.

## Conclusion

Redirect URI configuration is a precision exercise — every character matters. The rule is simple: the URI configured in Portainer's OAuth settings must be character-for-character identical to what's registered with your identity provider. Use the HTTPS URL that users see in their browser, always include the trailing slash, and for non-standard ports, include the port number.
