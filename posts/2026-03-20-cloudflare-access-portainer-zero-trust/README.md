# How to Configure Cloudflare Access with Portainer for Zero Trust

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Cloudflare Access, Zero Trust, Authentication, Security

Description: Learn how to put Portainer behind Cloudflare Access for zero-trust authentication, requiring identity verification before users can even see the Portainer login page.

## What Is Cloudflare Access?

Cloudflare Access sits in front of your application and authenticates users before forwarding requests to your backend. Even if someone finds your Portainer URL, they must first authenticate through Cloudflare - they never reach Portainer without valid identity.

```text
User → Cloudflare Access (identity check) → Cloudflare Tunnel → Portainer
         ↓ if not authenticated
         Cloudflare login page (Google, GitHub, etc.)
```

## Prerequisites

- Portainer accessible via Cloudflare Tunnel (see cloudflare-tunnel-portainer guide)
- Cloudflare Zero Trust account (free tier available)
- An identity provider (Google, GitHub, Okta, or email OTP)

## Step 1: Connect an Identity Provider

1. **Zero Trust → Settings → Authentication → Login methods**
2. Click **Add new** and select your provider:

```text
Google: Enter Client ID and Client Secret from Google OAuth
GitHub: Enter Client ID and Client Secret from GitHub OAuth app
OTP Email: No setup required - Cloudflare sends email codes
```

## Step 2: Create an Access Application

1. **Zero Trust → Access → Applications → Add an Application**
2. Select **Self-hosted**
3. Configure:

```text
Application name:    Portainer
Application domain:  portainer.yourdomain.com
Session duration:    24h (or your preference)
```

## Step 3: Create an Access Policy

Policies define who can access Portainer:

```text
Policy name:   Allow Team
Decision:      Allow

Include rules:
  Email domain: yourcompany.com    (anyone with this email)

  OR

  Email (list):
    alice@example.com
    bob@example.com
```

For stricter control, require a specific group:

```text
Include:
  Email domain: yourcompany.com
Require:
  Country: US    (only US-based users)
```

## Step 4: Enable App Launcher (Optional)

Add Portainer to the Cloudflare App Launcher so team members can find it at `yourdomain.cloudflareaccess.com`:

1. **Access → App Launcher → Enable**
2. The Portainer application will appear in the launcher automatically

## Step 5: Test Access

1. Open an incognito window
2. Visit `https://portainer.yourdomain.com`
3. You're redirected to Cloudflare's login page
4. Authenticate via your configured identity provider
5. After auth, you're forwarded to Portainer

## Service Tokens for Automation

For CI/CD pipelines or scripts that need to access Portainer's API:

1. **Access → Service Auth → Create Service Token**
2. Note the `CF-Access-Client-Id` and `CF-Access-Client-Secret`
3. Include in API requests:

```bash
curl -s "https://portainer.yourdomain.com/api/endpoints" \
  -H "CF-Access-Client-Id: ${CF_CLIENT_ID}" \
  -H "CF-Access-Client-Secret: ${CF_CLIENT_SECRET}" \
  -H "Authorization: Bearer ${PORTAINER_TOKEN}"
```

## Bypass Access for Specific Paths

If Portainer webhooks need to be accessible without auth:

1. Create a **Bypass** policy for the webhook path:

```text
Policy name:   Webhook Bypass
Decision:      Bypass

Include:
  Everyone

  Path: /api/webhooks/*
```

## WARP Tunnel for Team VPN Alternative

Instead of login-per-access, use Cloudflare WARP:

- Team members install the WARP client
- WARP connects them to your Zero Trust network
- Access policy: Require WARP device enrolled

## Conclusion

Cloudflare Access transforms Portainer's security posture by adding an identity layer that Portainer's built-in auth doesn't provide. Even compromised Portainer credentials don't help an attacker if they can't pass Cloudflare's identity check first. This is especially valuable for home labs and small teams that don't want to manage a VPN.
