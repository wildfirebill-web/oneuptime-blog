# How to Configure Ping Identity with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Authentication, SSO, Ping Identity, SAML

Description: Learn how to integrate Ping Identity (PingFederate or PingOne) with Rancher for SAML-based enterprise authentication.

Ping Identity is an enterprise identity management platform that provides SSO, MFA, and access management capabilities. Integrating Ping Identity with Rancher enables your organization to leverage existing Ping infrastructure for Kubernetes access management. This guide covers the SAML-based integration setup.

## Prerequisites

- Rancher v2.6 or later with admin access
- PingFederate or PingOne with admin access
- The Rancher server accessible via HTTPS
- Network connectivity between Rancher and Ping Identity servers

## Step 1: Gather Rancher SP Information

Collect the Rancher Service Provider details needed for Ping Identity:

```
Entity ID: https://rancher.example.com/v1-saml/ping/saml/metadata
ACS URL: https://rancher.example.com/v1-saml/ping/saml/acs
SLO URL: https://rancher.example.com/v1-saml/ping/saml/slo
```

Download the Rancher SP metadata:

```bash
curl -sk "https://rancher.example.com/v1-saml/ping/saml/metadata" > rancher-sp-metadata.xml
```

## Step 2: Create a SAML Connection in PingFederate

Configure Rancher as a Service Provider in PingFederate:

1. Log in to the PingFederate Admin Console.
2. Navigate to **SP Connections** and click **Create New**.
3. Select **Browser SSO Profiles** and **SAML 2.0**.

Import the Rancher metadata:

```
Connection Type: SP
Connection Template: Browser SSO
Partner's Entity ID: https://rancher.example.com/v1-saml/ping/saml/metadata
Connection Name: Rancher
```

Or manually enter:

```
Base URL: https://rancher.example.com
```

## Step 3: Configure Browser SSO

Set up the Browser SSO profile:

1. Under **Browser SSO**, click **Configure**.
2. Select the SAML profiles:

```
☑ SP-Initiated SSO
☑ SP-Initiated SLO
☑ IdP-Initiated SSO
```

3. Configure the assertion lifetime:

```
Minutes Before: 5
Minutes After: 5
```

## Step 4: Configure Attribute Contracts

Define the SAML attributes to send to Rancher:

1. Under **Attribute Contract**, extend the default contract with:

```
Attribute Name: displayName
Attribute Format: urn:oasis:names:tc:SAML:2.0:attrname-format:basic

Attribute Name: userName
Attribute Format: urn:oasis:names:tc:SAML:2.0:attrname-format:basic

Attribute Name: uid
Attribute Format: urn:oasis:names:tc:SAML:2.0:attrname-format:basic

Attribute Name: groups
Attribute Format: urn:oasis:names:tc:SAML:2.0:attrname-format:basic

Attribute Name: email
Attribute Format: urn:oasis:names:tc:SAML:2.0:attrname-format:basic
```

## Step 5: Map Attributes from the Identity Store

Configure the attribute source and mapping:

1. Under **Authentication Source Mapping**, select your identity store (LDAP, AD, or database).
2. Map the attributes:

```
SAML Attribute    -> Source Attribute
displayName       -> displayName (from LDAP)
userName          -> sAMAccountName (from AD) or uid (from LDAP)
uid               -> sAMAccountName (from AD) or uid (from LDAP)
email             -> mail
groups            -> memberOf
```

For group attribute mapping, configure a custom data source:

```
Type: LDAP
Base DN: ou=groups,dc=example,dc=com
Filter: (member=${SAML_SUBJECT})
Attribute: cn
```

## Step 6: Configure Protocol Settings

Set the protocol-specific options:

```
Assertion Consumer Service URL: https://rancher.example.com/v1-saml/ping/saml/acs
Binding: POST
SLO Service URL: https://rancher.example.com/v1-saml/ping/saml/slo
SLO Binding: POST
Signature Policy: Sign Response and Assertion
Signature Algorithm: RSA-SHA256
```

## Step 7: Download PingFederate Metadata

Export the PingFederate IdP metadata:

```bash
# Download the metadata
curl -s "https://pingfederate.example.com/pf/federation_metadata.ping?PartnerSpId=https://rancher.example.com/v1-saml/ping/saml/metadata" \
  > ping-idp-metadata.xml

# Or get the general metadata
curl -s "https://pingfederate.example.com/pf/federation_metadata.ping" \
  > ping-idp-metadata.xml
```

Or export from the PingFederate console:

1. Go to the SP Connection for Rancher.
2. Under **Metadata**, click **Export**.

## Step 8: Configure Ping Identity in Rancher

Set up the Ping auth provider in Rancher:

1. Log in to Rancher as an administrator.
2. Navigate to **Users & Authentication** then **Auth Provider**.
3. Select **Ping**.

Enter the configuration:

```
Display Name Field: displayName
User Name Field: userName
UID Field: uid
Groups Field: groups
Entity ID: https://rancher.example.com/v1-saml/ping/saml/metadata
Rancher API Host: https://rancher.example.com
IdP Metadata: (paste the PingFederate metadata XML)
```

## Step 9: Test and Enable

Test the Ping Identity integration:

1. Click **Test** in the Rancher configuration.
2. Authenticate through the PingFederate login page.
3. Verify the returned user information and group memberships.

If testing succeeds, click **Enable** to activate Ping Identity authentication.

If testing fails, check common issues:

```bash
# Check Rancher logs
kubectl logs -l app=rancher -n cattle-system --tail=200 | grep -i "saml\|ping\|auth"
```

| Issue | Solution |
|-------|----------|
| Connection refused | Verify network connectivity to PingFederate |
| Invalid signature | Check that the signing certificate matches |
| Missing attributes | Review the attribute contract and mapping in PingFederate |
| Clock skew | Sync NTP on both Rancher and PingFederate servers |

## Step 10: Assign Roles to Ping Identity Groups

Map Ping Identity groups to Rancher roles:

1. Navigate to **Users & Authentication** then **Groups**.
2. Search for groups that were returned in the SAML assertion.
3. Assign roles.

```
Group: Platform-Admins -> Administrator
Group: DevOps -> Standard User
Group: Developers -> Standard User
```

For cluster and project-level access, navigate to the respective member sections and add groups with the appropriate roles.

## PingOne Configuration

If using PingOne instead of PingFederate:

1. Log in to the PingOne Admin Console.
2. Navigate to **Connections** then **Applications**.
3. Click **Add Application** and select **SAML**.
4. Enter the Rancher details:

```
Application Name: Rancher
ACS URL: https://rancher.example.com/v1-saml/ping/saml/acs
Entity ID: https://rancher.example.com/v1-saml/ping/saml/metadata
SLO Endpoint: https://rancher.example.com/v1-saml/ping/saml/slo
```

5. Configure attribute mappings under the **Attribute Mappings** tab.
6. Download the PingOne metadata from the **Configuration** tab.

## Best Practices

- **Use signed assertions**: Always enable signing for both SAML responses and assertions.
- **Limit group claims**: Filter the groups sent in SAML assertions to only include Rancher-relevant groups to reduce assertion size.
- **Configure session timeouts**: Align session timeouts between PingFederate and Rancher for a consistent user experience.
- **Monitor authentication**: Use PingFederate's audit logging to track authentication events for the Rancher connection.
- **Plan for certificate rotation**: Track the PingFederate signing certificate expiration and update Rancher's configuration before it expires.

## Conclusion

Ping Identity integration with Rancher brings enterprise-grade identity management to your Kubernetes platform. Whether using PingFederate or PingOne, the SAML-based integration provides secure SSO, centralized access control, and compatibility with your existing identity infrastructure. Follow the attribute mapping steps carefully and test thoroughly to ensure a smooth rollout to your organization.
