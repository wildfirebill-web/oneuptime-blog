# How to Configure SAML Authentication in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Authentication, SAML, SSO

Description: A practical guide to configuring SAML-based single sign-on authentication in Rancher with any SAML 2.0 identity provider.

SAML (Security Assertion Markup Language) enables single sign-on (SSO) authentication between Rancher and your identity provider. This allows users to authenticate once with their corporate identity provider and gain access to Rancher without entering separate credentials. This guide covers the general SAML setup process.

## Prerequisites

- Rancher v2.6 or later with admin access
- A SAML 2.0 compatible identity provider (IdP)
- Access to configure your identity provider
- The Rancher server URL must be accessible from users' browsers
- TLS configured on the Rancher server

## Understanding SAML Flow

The SAML authentication flow works as follows:

1. User navigates to the Rancher login page.
2. User clicks the SAML login button.
3. Browser redirects to the identity provider.
4. User authenticates with the IdP.
5. IdP sends a SAML assertion back to Rancher.
6. Rancher validates the assertion and creates a session.

## Step 1: Get Rancher SP Metadata

First, obtain the Rancher Service Provider (SP) metadata:

1. Log in to Rancher as an administrator.
2. Navigate to **Users & Authentication** then **Auth Provider**.
3. Select your SAML provider (e.g., **Ping**, **Keycloak**, **ADFS**, or **Shibboleth**).
4. Note the SP metadata URL, typically:

```plaintext
https://rancher.example.com/v1-saml/adfs/saml/metadata
```

Or download the metadata:

```bash
curl -sk "https://rancher.example.com/v1-saml/adfs/saml/metadata" > rancher-sp-metadata.xml
```

## Step 2: Configure Your Identity Provider

Register Rancher as a Service Provider in your IdP. The general process involves:

1. Import or manually enter the Rancher SP metadata.
2. Configure the following endpoints:

```plaintext
Entity ID: https://rancher.example.com/v1-saml/adfs/saml/metadata
ACS URL: https://rancher.example.com/v1-saml/adfs/saml/acs
Single Logout URL: https://rancher.example.com/v1-saml/adfs/saml/slo
```

3. Configure attribute mappings to include:

```plaintext
UID Attribute: uid (or preferred username attribute)
Display Name Attribute: displayName
User Name Attribute: userName
Groups Attribute: groups
Email Attribute: email
```

## Step 3: Configure Attribute Statements

Set up SAML attribute statements in your IdP to send the required user information:

```xml
<!-- Example SAML attribute statement configuration -->
<AttributeStatement>
  <Attribute Name="uid">
    <AttributeValue>john.doe</AttributeValue>
  </Attribute>
  <Attribute Name="displayName">
    <AttributeValue>John Doe</AttributeValue>
  </Attribute>
  <Attribute Name="email">
    <AttributeValue>john.doe@example.com</AttributeValue>
  </Attribute>
  <Attribute Name="groups">
    <AttributeValue>developers</AttributeValue>
    <AttributeValue>devops</AttributeValue>
  </Attribute>
</AttributeStatement>
```

## Step 4: Obtain IdP Metadata

Get the identity provider metadata XML:

```bash
# Download IdP metadata (example for ADFS)
curl -sk "https://adfs.example.com/FederationMetadata/2007-06/FederationMetadata.xml" \
  > idp-metadata.xml

# Or for Shibboleth
curl -sk "https://idp.example.com/idp/shibboleth" > idp-metadata.xml
```

Extract the key information from the metadata:

```plaintext
IdP Entity ID: https://adfs.example.com/adfs/services/trust
SSO URL: https://adfs.example.com/adfs/ls/
SLO URL: https://adfs.example.com/adfs/ls/?wa=wsignout1.0
Certificate: (the IdP signing certificate)
```

## Step 5: Configure SAML in Rancher

Enter the IdP details in Rancher:

1. Navigate to **Users & Authentication** then **Auth Provider**.
2. Select your SAML provider.
3. Fill in the configuration:

```plaintext
Display Name Field: displayName
User Name Field: userName
UID Field: uid
Groups Field: groups
Entity ID Field: https://rancher.example.com/v1-saml/adfs/saml/metadata
Rancher API Host: https://rancher.example.com
IDP Metadata: (paste the IdP metadata XML)
```

Or enter the fields manually:

```plaintext
IdP Entity ID: https://adfs.example.com/adfs/services/trust
SAML SSO URL: https://adfs.example.com/adfs/ls/
IDP Certificate: (paste the IdP signing certificate in PEM format)
```

## Step 6: Configure Group Mappings

Map SAML groups to Rancher roles:

```plaintext
SAML Group Attribute Name: groups

Group Mappings:
  "platform-admins" -> Administrator
  "developers" -> Standard User
  "readonly-users" -> User-Base
```

In the Rancher UI:

1. Go to **Users & Authentication** then **Groups**.
2. Search for a SAML group.
3. Assign the appropriate global role.

## Step 7: Test SAML Authentication

Test the configuration before enabling it for all users:

1. Click the **Test** button in the SAML configuration page.
2. You will be redirected to your IdP login page.
3. Authenticate with your IdP credentials.
4. Verify that you are redirected back to Rancher with proper user information.

If testing fails, check:

```bash
# Check Rancher logs for SAML errors
kubectl logs -l app=rancher -n cattle-system --tail=200 | grep -i "saml\|auth"
```

Common SAML errors:

| Error | Cause | Solution |
|-------|-------|----------|
| Invalid signature | Certificate mismatch | Update the IdP certificate in Rancher |
| Audience mismatch | Wrong Entity ID | Ensure Entity ID matches in both Rancher and IdP |
| Clock skew | Time difference between servers | Sync NTP on both Rancher and IdP servers |
| Missing attributes | Attribute mapping error | Verify attribute names match between IdP and Rancher |

## Step 8: Enable SAML Authentication

Once testing succeeds:

1. Click **Enable** to activate SAML authentication.
2. Confirm the action.

The Rancher login page will now show a SAML SSO login button.

## Step 9: Configure Single Logout

Set up Single Logout (SLO) so that logging out of Rancher also logs the user out of the IdP:

1. In your IdP, configure the SLO endpoint:

```plaintext
SLO URL: https://rancher.example.com/v1-saml/adfs/saml/slo
SLO Binding: HTTP-Redirect
```

2. In Rancher, verify the SLO configuration is enabled.

Test SLO by logging out of Rancher and verifying you are also logged out of the IdP.

## Step 10: Maintain the SAML Integration

Ongoing maintenance tasks:

### Certificate Rotation

When your IdP certificate is about to expire:

1. Generate a new certificate in your IdP.
2. Update the certificate in Rancher's SAML configuration.
3. Test authentication with the new certificate.
4. Remove the old certificate from the IdP.

### Monitor Authentication

```bash
# Check for SAML authentication failures
kubectl logs -l app=rancher -n cattle-system --tail=500 | \
  grep -i "saml.*error\|saml.*fail"

# Count successful SAML logins
kubectl logs -l app=rancher -n cattle-system --tail=1000 | \
  grep -i "saml.*success" | wc -l
```

## Best Practices

- **Use HTTPS everywhere**: Both Rancher and the IdP must use HTTPS for SAML to function securely.
- **Sync clocks**: Ensure NTP is configured on both Rancher and IdP servers to prevent clock skew issues.
- **Map groups, not individuals**: Use group-based role assignments for scalable access management.
- **Plan for certificate rotation**: Track IdP certificate expiration dates and rotate well in advance.
- **Test thoroughly**: Test SAML login, logout, and group-based access with multiple users before rolling out to the organization.

## Conclusion

SAML authentication in Rancher provides enterprise-grade single sign-on for your Kubernetes management platform. By integrating with your organization's identity provider, you eliminate the need for separate Rancher credentials and leverage your existing identity governance policies. Follow the configuration steps carefully, test extensively, and maintain your certificates to ensure a smooth authentication experience for all users.
