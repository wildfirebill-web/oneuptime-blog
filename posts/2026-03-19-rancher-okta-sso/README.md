# How to Configure Okta SSO with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Authentication, SSO, Okta, SAML

Description: Step-by-step instructions for integrating Okta single sign-on with Rancher using SAML 2.0 for enterprise authentication.

Okta is a leading identity management platform used by many organizations. Integrating Okta with Rancher allows your teams to use their Okta credentials for Kubernetes cluster access through SAML-based single sign-on. This guide provides detailed steps for setting up the integration.

## Prerequisites

- Rancher v2.6 or later with admin access
- An Okta organization with admin privileges
- The Rancher server accessible via HTTPS
- Okta admin access to create SAML applications

## Step 1: Create a SAML Application in Okta

Set up the Rancher application in Okta:

1. Log in to the Okta Admin Console.
2. Navigate to **Applications** then **Applications**.
3. Click **Create App Integration**.
4. Select **SAML 2.0** and click **Next**.

Enter the general settings:

```plaintext
App Name: Rancher
App Logo: (upload the Rancher logo if desired)
App Visibility: Do not display application icon to users (optional)
```

Click **Next** to proceed to SAML settings.

## Step 2: Configure SAML Settings in Okta

Enter the Rancher SAML endpoints:

```plaintext
Single sign-on URL: https://rancher.example.com/v1-saml/okta/saml/acs
Audience URI (SP Entity ID): https://rancher.example.com/v1-saml/okta/saml/metadata
Default RelayState: (leave blank)
Name ID format: Unspecified
Application username: Okta username
```

Configure the response and assertion:

```plaintext
Response: Signed
Assertion Signature: Signed
Signature Algorithm: RSA-SHA256
Digest Algorithm: SHA256
Assertion Encryption: Unencrypted
```

## Step 3: Configure Attribute Statements

Add the following attribute statements in Okta:

```plaintext
Name: displayName
Name Format: Unspecified
Value: user.displayName

Name: userName
Name Format: Unspecified
Value: user.login

Name: uid
Name Format: Unspecified
Value: user.login

Name: email
Name Format: Unspecified
Value: user.email
```

Add group attribute statements:

```plaintext
Name: groups
Name Format: Unspecified
Filter: Matches regex: .*
```

This sends all groups the user belongs to. You can narrow the filter to only send specific groups:

```plaintext
Filter: Starts with: rancher-
```

Click **Next** and then **Finish** to create the application.

## Step 4: Download Okta IdP Metadata

Get the Okta metadata for Rancher:

1. In the Okta Admin Console, go to the Rancher application.
2. Click the **Sign On** tab.
3. Find the **Metadata URL** link and copy it.

```bash
# Download the metadata
curl -s "https://your-org.okta.com/app/xxxxx/sso/saml/metadata" > okta-metadata.xml
```

Or from the Sign On tab, click **View SAML setup instructions** to get:

- Identity Provider Single Sign-On URL
- Identity Provider Issuer
- X.509 Certificate

## Step 5: Assign Users and Groups in Okta

Assign users to the Rancher application:

1. In the Rancher application in Okta, click the **Assignments** tab.
2. Click **Assign** and select **Assign to Groups**.
3. Search for and assign the relevant groups.

```yaml
Assigned Groups:
  - Everyone (or specific groups)
  - DevOps-Team
  - Platform-Engineers
  - Developers
```

## Step 6: Configure Rancher for Okta SAML

Now configure Rancher to use Okta:

1. Log in to Rancher as an administrator.
2. Navigate to **Users & Authentication** then **Auth Provider**.
3. Select **Okta** from the list.

Enter the configuration details:

```plaintext
Display Name Field: displayName
User Name Field: userName
UID Field: uid
Groups Field: groups
Entity ID: https://rancher.example.com/v1-saml/okta/saml/metadata
Rancher API Host: https://rancher.example.com
IdP Metadata: (paste the downloaded Okta metadata XML)
```

Or manually enter:

```plaintext
IdP Entity ID: http://www.okta.com/xxxxx
SAML SSO URL: https://your-org.okta.com/app/xxxxx/sso/saml
IDP Certificate: (paste the Okta X.509 certificate)
```

## Step 7: Test the Integration

Test the Okta SSO configuration:

1. Click the **Test** button in Rancher.
2. You will be redirected to the Okta login page.
3. Enter your Okta credentials.
4. Verify you are redirected back to Rancher with correct user information.

Check that the returned attributes include:

- Display name
- Username
- Groups

If the test fails:

```bash
# Check Rancher logs
kubectl logs -l app=rancher -n cattle-system --tail=200 | grep -i "saml\|okta"
```

## Step 8: Enable Okta Authentication

After successful testing:

1. Click **Enable** to activate Okta SSO.
2. Confirm the action.

The Rancher login page now shows a **Log in with Okta** button.

## Step 9: Map Okta Groups to Rancher Roles

Configure role mappings for Okta groups:

1. Navigate to **Users & Authentication** then **Groups**.
2. Search for Okta groups.
3. Assign Rancher roles.

Global role mappings:

```plaintext
Okta Group: Platform-Engineers -> Administrator
Okta Group: DevOps-Team -> Standard User
Okta Group: Developers -> Standard User
Okta Group: Auditors -> User-Base (read-only)
```

Cluster-level mappings:

1. Navigate to a specific cluster.
2. Go to **Cluster Members** then **Add**.
3. Search for the Okta group.
4. Assign the cluster role.

```plaintext
Okta Group: App-Team-Backend -> Cluster Member (production-cluster)
Okta Group: App-Team-Frontend -> Cluster Member (production-cluster)
Okta Group: QA-Team -> Cluster Member (staging-cluster)
```

## Step 10: Configure Advanced Okta Settings

Fine-tune your Okta integration:

### Session Duration

Configure how long Rancher sessions last:

```bash
# Set auth token max TTL (in minutes)
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "960"}' \
  "https://rancher.example.com/v3/settings/auth-token-max-ttl-minutes"
```

### Okta MFA Integration

If Okta MFA is configured, it works transparently with Rancher. When users authenticate through Okta, they will be prompted for MFA as configured in your Okta policies. No additional configuration is needed in Rancher.

### Restrict Access by Okta Group

To limit Rancher access to specific Okta groups, configure an Okta sign-on policy:

1. In the Okta Admin Console, go to the Rancher application.
2. Click the **Sign On** tab.
3. Add a sign-on policy rule that restricts access to specific groups.

## Troubleshooting

Common issues with Okta SSO:

| Issue | Solution |
|-------|----------|
| SAML response invalid | Verify the ACS URL matches exactly in both Okta and Rancher |
| Groups not appearing | Check the group attribute statement filter in Okta |
| Clock skew error | Ensure NTP is configured on the Rancher server |
| Certificate error after Okta cert rotation | Update the IdP certificate in Rancher's Okta configuration |
| User gets 403 after login | Assign global or cluster roles to the user's Okta groups |

## Best Practices

- **Use group-based access**: Manage Rancher access through Okta groups rather than individual user assignments.
- **Implement Okta MFA**: Enable multi-factor authentication in Okta for an additional layer of security.
- **Monitor SSO usage**: Use Okta's reporting to track authentication activity for the Rancher application.
- **Plan for certificate rotation**: Okta certificates have expiration dates. Set reminders to update them before they expire.
- **Test with multiple users**: Verify that users from different groups receive the correct Rancher roles.

## Conclusion

Integrating Okta SSO with Rancher provides a seamless authentication experience for your Kubernetes management platform. By leveraging Okta's identity management capabilities, you gain centralized access control, multi-factor authentication, and detailed audit logging. Follow the steps in this guide to establish a secure and user-friendly SSO experience, and leverage Okta groups for scalable role management in Rancher.
