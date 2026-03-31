# How to Set Up SSO for ClickHouse Cloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Cloud, SSO, SAML, Identity Provider, Security, Authentication

Description: Learn how to configure Single Sign-On for ClickHouse Cloud using SAML 2.0 with identity providers like Okta, Azure AD, and Google Workspace.

---

ClickHouse Cloud supports SAML 2.0 Single Sign-On, allowing your team to authenticate using your existing identity provider (IdP). SSO centralizes access control, enforces MFA policies, and eliminates the need for separate ClickHouse Cloud passwords.

## Supported Identity Providers

- Okta
- Microsoft Entra ID (formerly Azure AD)
- Google Workspace
- Any SAML 2.0 compatible IdP

## Prerequisites

- ClickHouse Cloud organization on Scale or Enterprise tier
- Admin access to your identity provider
- ClickHouse Cloud organization admin role

## Step 1 - Get ClickHouse SAML Metadata

In the ClickHouse Cloud console:
1. Go to "Organization Settings" - "Security" - "Single Sign-On"
2. Copy the SAML SP metadata URL or download the metadata XML

```text
ACS URL: https://auth.clickhouse.cloud/login/callback?connection=org-<your-org-id>
Entity ID: urn:auth0:clickhouse-cloud:org-<your-org-id>
```

## Step 2 - Configure Okta

1. In Okta, create a new SAML 2.0 application
2. Set Single Sign-On URL to the ClickHouse ACS URL
3. Set Audience URI (SP Entity ID) to the ClickHouse Entity ID
4. Configure attribute statements:

```text
email   -> user.email
name    -> user.firstName + " " + user.lastName
```

5. Download the Okta IdP metadata XML

## Step 3 - Configure Azure AD / Microsoft Entra

1. In Entra ID, create an Enterprise Application
2. Set up SAML with the ClickHouse ACS URL and Entity ID
3. Map claims:

```text
emailaddress -> user.mail
name         -> user.displayname
```

## Step 4 - Upload IdP Metadata to ClickHouse Cloud

In the ClickHouse Cloud console, upload the IdP metadata XML or enter the configuration manually:

```text
IdP Entity ID: https://idp.example.com/issuer
IdP SSO URL:   https://idp.example.com/sso/saml
X.509 Certificate: (paste your IdP certificate)
```

## Step 5 - Test SSO Login

Use the "Test SSO" button in the console before enabling it organization-wide. Verify that:
- Users can log in via SSO
- Email addresses match your ClickHouse Cloud user accounts
- Roles are assigned correctly

## Enforcing SSO

Once SSO is configured and tested, enable "Enforce SSO" to require all organization members to authenticate via your IdP. This disables password-based login for the organization.

## Summary

ClickHouse Cloud SSO uses SAML 2.0 to integrate with Okta, Microsoft Entra, Google Workspace, and other IdPs. Configure the SP metadata in your IdP, upload the IdP metadata to ClickHouse Cloud, test the integration, and then enforce SSO to centralize authentication and leverage your existing MFA and access policies.
