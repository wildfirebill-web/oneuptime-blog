# How to Configure NeuVector SAML SSO

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, SAML, SSO, Authentication, Identity Management

Description: Configure SAML 2.0 Single Sign-On for NeuVector to enable your team to authenticate using Okta, Azure AD, or other identity providers.

## Introduction

SAML 2.0 (Security Assertion Markup Language) Single Sign-On allows NeuVector users to authenticate using your organization's identity provider (IdP). This eliminates the need for separate NeuVector passwords and provides centralized access control through your existing SSO infrastructure. This guide covers configuration with Okta and Azure AD.

## Prerequisites

- NeuVector Manager accessible via HTTPS
- An identity provider that supports SAML 2.0 (Okta, Azure AD, Google Workspace, etc.)
- Admin access to both NeuVector and your IdP
- NeuVector Manager URL (e.g., `https://neuvector.company.com:8443`)

## Step 1: Get NeuVector SAML Metadata

First, gather NeuVector's SAML service provider information:

```bash
# NeuVector SAML Service Provider details

# These values are needed when configuring your IdP:

Entity ID (Audience URI): https://neuvector.company.com:8443
ACS URL (Reply URL): https://neuvector.company.com:8443/v1/token_auth_server/saml
```

## Step 2: Configure Okta as the IdP

### Create an Okta Application

1. Log in to Okta Admin Console
2. Go to **Applications** > **Applications**
3. Click **Create App Integration**
4. Select **SAML 2.0**
5. Configure the app:

```text
App name: NeuVector Security Platform
App logo: (upload NeuVector logo)
```

6. In SAML Settings:

```text
Single Sign On URL (ACS URL): https://neuvector.company.com:8443/v1/token_auth_server/saml
Audience URI (SP Entity ID): https://neuvector.company.com:8443
Name ID format: EmailAddress
Application username: Email

Attribute Statements:
- Name: username
  Value: user.login
- Name: email
  Value: user.email
- Name: firstname
  Value: user.firstName

Group Attribute Statements:
- Name: groups
  Filter: Starts with: NeuVector-
```

7. Download the IdP metadata XML or note:
   - Identity Provider SSO URL
   - Identity Provider Issuer
   - X.509 Certificate

### Configure NeuVector with Okta Settings

```bash
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/system/config" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "auth_order": ["saml", "local"],
      "saml_config": {
        "sso_url": "https://dev-123456.okta.com/app/neuvector/sso/saml",
        "issuer": "http://www.okta.com/abc123def456",
        "x509_cert": "MIIDpDCCAoygAwIBAgIGAXXXXXXXXXXXX...",
        "username_claim": "username",
        "email_claim": "email",
        "group_claim": "groups",
        "group_mapped_roles": [
          {
            "group": "NeuVector-Admins",
            "global_role": "admin"
          },
          {
            "group": "NeuVector-SecurityTeam",
            "global_role": "reader"
          },
          {
            "group": "NeuVector-Developers",
            "global_role": "",
            "role_domains": {
              "reader": ["development"]
            }
          }
        ],
        "redirect_url": "https://neuvector.company.com:8443",
        "enable": true,
        "default_role": ""
      }
    }
  }'
```

## Step 3: Configure Azure AD as the IdP

### Create an Azure AD Enterprise Application

```bash
# Using Azure CLI

# Create the enterprise application
az ad app create \
  --display-name "NeuVector" \
  --reply-urls "https://neuvector.company.com:8443/v1/token_auth_server/saml"

# Get the application ID
APP_ID=$(az ad app list --display-name "NeuVector" --query "[].appId" -o tsv)
echo "Application ID: ${APP_ID}"
```

In Azure Portal:
1. Go to **Azure Active Directory** > **Enterprise Applications**
2. Click **New application** > **Create your own application**
3. Name it "NeuVector" and select **Non-gallery application**
4. Go to **Single sign-on** > **SAML**
5. Configure Basic SAML Configuration:

```text
Identifier (Entity ID): https://neuvector.company.com:8443
Reply URL (ACS URL): https://neuvector.company.com:8443/v1/token_auth_server/saml
Sign on URL: https://neuvector.company.com:8443
```

6. Configure User Attributes & Claims:

```text
Unique User Identifier: user.mail
Additional claims:
- username: user.userprincipalname
- email: user.mail
- groups: user.groups (group Object IDs)
```

7. Download the Federation Metadata XML or note the values

### Configure NeuVector with Azure AD Settings

```bash
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/system/config" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "saml_config": {
        "sso_url": "https://login.microsoftonline.com/<tenant-id>/saml2",
        "issuer": "https://sts.windows.net/<tenant-id>/",
        "x509_cert": "MIIC8DCCAdigAwIBAgIQXXXXXXXXXX...",
        "username_claim": "username",
        "email_claim": "email",
        "group_claim": "groups",
        "group_mapped_roles": [
          {
            "group": "<azure-ad-group-object-id-for-admins>",
            "global_role": "admin"
          },
          {
            "group": "<azure-ad-group-object-id-for-readers>",
            "global_role": "reader"
          }
        ],
        "enable": true
      }
    }
  }'
```

## Step 4: Test SAML SSO Login

```bash
# Access the NeuVector UI via browser
# Navigate to: https://neuvector.company.com:8443
# Click "Login with SSO"
# Should redirect to your IdP login page

# After authentication, verify the token contains the correct role
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/token_auth_server/saml" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "SAMLResponse=$(cat saml-response.xml | base64)" | \
  jq '.token | {username: .username, role: .role}'
```

## Step 5: Configure Default Role for Unmatched Groups

Set a default role for users who don't match any group mapping:

```bash
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/system/config" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "saml_config": {
        "default_role": "reader"
      }
    }
  }'
```

## Conclusion

SAML SSO integration provides enterprise-grade authentication for NeuVector, enabling your team to use existing corporate credentials and benefiting from centralized access revocation. When an employee leaves, removing them from the IdP immediately revokes their NeuVector access. Always maintain a local admin fallback account with a secure password for emergencies when the IdP is unavailable.
