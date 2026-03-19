# How to Enable Two-Factor Authentication in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Authentication, 2FA, RBAC

Description: Learn how to implement two-factor authentication for Rancher access using identity provider integrations and additional security layers.

Two-factor authentication (2FA) adds an essential security layer to your Rancher environment by requiring users to provide a second form of verification beyond their password. While Rancher does not have a built-in 2FA feature for local accounts, you can implement 2FA effectively through identity provider integrations. This guide covers multiple approaches to enabling 2FA for Rancher.

## Prerequisites

- Rancher v2.6 or later with admin access
- An external authentication provider configured (or willingness to set one up)
- Users with smartphones or hardware tokens for TOTP
- Admin access to your identity provider

## Understanding 2FA Options for Rancher

Rancher relies on its authentication providers for 2FA enforcement. The recommended approaches are:

1. **OIDC/SAML provider with built-in MFA** (Okta, Azure AD, Keycloak, etc.)
2. **GitHub with 2FA enforcement**
3. **Proxy-based authentication with MFA**

## Method 1: 2FA via Keycloak

Keycloak provides built-in TOTP-based 2FA that applies to Rancher logins.

### Step 1: Configure OTP Policy in Keycloak

1. Log in to the Keycloak Admin Console.
2. Navigate to **Authentication** in your realm.
3. Click **Policies** then **OTP Policy**.

Configure the OTP settings:

```plaintext
OTP Type: Time-Based (TOTP)
OTP Hash Algorithm: SHA1
Number of Digits: 6
Look Ahead Window: 1
OTP Token Period: 30 seconds
```

### Step 2: Enable OTP as Required Action

1. Go to **Authentication** then **Required Actions**.
2. Find **Configure OTP** in the list.
3. Set it as **Default Action** so new users must configure it on first login.

```plaintext
Configure OTP:
  Enabled: ON
  Default Action: ON
```

### Step 3: Create a Browser Flow with OTP

1. Go to **Authentication** then **Flows**.
2. Duplicate the **Browser** flow.
3. Add an OTP execution step:

```plaintext
Browser Flow (copy):
  ├── Cookie (Alternative)
  ├── Identity Provider Redirector (Alternative)
  └── Forms (Alternative)
      ├── Username Password Form (Required)
      └── OTP Form (Required)
```

4. Bind this flow to the browser flow:
   - Go to **Authentication** then **Bindings**.
   - Set **Browser Flow** to your custom flow.

### Step 4: Test OTP with Rancher

1. Log out of Rancher.
2. Click **Log in with Keycloak**.
3. Enter your username and password.
4. You will be prompted to set up TOTP if it is your first time.
5. Scan the QR code with an authenticator app (Google Authenticator, Authy, etc.).
6. Enter the verification code.
7. Verify that you are logged in to Rancher.

## Method 2: 2FA via Azure AD Conditional Access

### Step 1: Create a Conditional Access Policy

1. Go to the Azure Portal and navigate to **Azure AD** then **Security** then **Conditional Access**.
2. Click **New policy**.

Configure the policy:

```plaintext
Name: Rancher MFA Requirement
Assignments:
  Users: All users (or specific groups)
  Cloud apps: Select the Rancher app registration
Conditions:
  (configure as needed - e.g., all locations)
Grant:
  Grant access
  ☑ Require multi-factor authentication
Session:
  Sign-in frequency: 8 hours
```

3. Enable the policy and click **Create**.

### Step 2: Configure Azure MFA Methods

1. Navigate to **Azure AD** then **Security** then **Authentication methods**.
2. Enable the MFA methods your organization supports:

```plaintext
☑ Microsoft Authenticator (push notifications)
☑ OATH hardware tokens
☑ OATH software tokens (TOTP apps)
☑ SMS (less secure, not recommended)
☑ Voice call (less secure, not recommended)
```

### Step 3: Verify MFA with Rancher

1. Navigate to Rancher and click **Log in with Azure AD**.
2. Enter your Azure AD credentials.
3. Complete the MFA challenge.
4. Verify successful login to Rancher.

## Method 3: 2FA via Okta

### Step 1: Configure Okta MFA Policy

1. Log in to the Okta Admin Console.
2. Navigate to **Security** then **Multifactor**.
3. Enable MFA factors:

```plaintext
Factor Types:
  ☑ Okta Verify (push/TOTP)
  ☑ Google Authenticator
  ☑ YubiKey (hardware token)
  ☐ SMS (disabled for security)
  ☐ Voice Call (disabled for security)
```

### Step 2: Create an MFA Sign-On Policy

1. Navigate to the Rancher application in Okta.
2. Click the **Sign On** tab.
3. Add a sign-on rule:

```plaintext
Rule Name: Require MFA for Rancher
Conditions:
  When user signs in: All users
  After MFA enrollment: Challenge at every sign on
Access:
  Authentication policy: Any 2 factor types
```

### Step 3: Enroll Users

Users will be prompted to enroll in MFA on their next login attempt:

1. Navigate to Rancher.
2. Click **Log in with Okta**.
3. Enter credentials.
4. Follow the enrollment flow to set up an authenticator app.
5. Complete the MFA challenge.

## Method 4: 2FA via GitHub Organization

### Step 1: Enforce 2FA in GitHub Organization

1. Go to your GitHub organization settings.
2. Navigate to **Authentication security**.
3. Enable **Require two-factor authentication**:

```plaintext
☑ Require two-factor authentication for everyone in the organization
```

4. Set a grace period for existing members to enable 2FA.

### Step 2: Verify Enforcement

Members who do not enable 2FA within the grace period will be removed from the organization automatically. This ensures that all Rancher logins through GitHub have 2FA enabled.

## Method 5: Reverse Proxy with MFA

For environments using local authentication, add an MFA layer with a reverse proxy:

### Step 1: Deploy OAuth2 Proxy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oauth2-proxy
  namespace: cattle-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: oauth2-proxy
  template:
    metadata:
      labels:
        app: oauth2-proxy
    spec:
      containers:
        - name: oauth2-proxy
          image: quay.io/oauth2-proxy/oauth2-proxy:latest
          args:
            - --provider=oidc
            - --oidc-issuer-url=https://keycloak.example.com/realms/your-realm
            - --client-id=rancher-proxy
            - --client-secret=$(CLIENT_SECRET)
            - --cookie-secret=$(COOKIE_SECRET)
            - --upstream=https://rancher-internal.example.com
            - --email-domain=example.com
            - --pass-access-token=true
            - --set-xauthrequest=true
          ports:
            - containerPort: 4180
```

### Step 2: Configure Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rancher-mfa
  annotations:
    nginx.ingress.kubernetes.io/auth-url: "http://oauth2-proxy.cattle-system.svc.cluster.local:4180/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://rancher.example.com/oauth2/start"
spec:
  rules:
    - host: rancher.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rancher
                port:
                  number: 443
```

## Monitoring 2FA Compliance

Track which users have 2FA enabled:

```bash
# For GitHub-based auth, check organization members
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/orgs/your-org/members?filter=2fa_disabled"

# For Rancher, audit authentication events
kubectl logs -l app=rancher -n cattle-system --tail=500 | \
  grep -i "login\|auth" | tail -20
```

## Best Practices

- **Use phishing-resistant methods**: Prefer authenticator apps and hardware tokens over SMS-based 2FA.
- **Enforce MFA at the IdP level**: Configure MFA in your identity provider so it applies to all applications, not just Rancher.
- **Provide backup codes**: Ensure users have recovery options if they lose their 2FA device.
- **Audit MFA enrollment**: Regularly check that all users have MFA enabled and no exceptions exist.
- **Test the recovery flow**: Verify that the account recovery process works before users need it.

## Conclusion

While Rancher does not include built-in 2FA, you can effectively implement it through identity provider integrations. Whether you use Keycloak, Azure AD, Okta, or GitHub, the key is to enforce MFA at the identity provider level so that every Rancher authentication goes through a second factor. Choose the approach that aligns with your existing identity infrastructure and security requirements.
