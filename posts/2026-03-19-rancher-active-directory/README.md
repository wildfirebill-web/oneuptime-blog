# How to Configure Active Directory Authentication in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Authentication, Active Directory, RBAC

Description: A step-by-step guide to integrating Microsoft Active Directory with Rancher for centralized user authentication and access management.

Microsoft Active Directory (AD) is one of the most widely used directory services in enterprise environments. Integrating AD with Rancher allows your teams to use their existing corporate credentials to access Kubernetes clusters. This guide walks through the complete setup process.

## Prerequisites

- Rancher v2.6 or later with admin access
- A Microsoft Active Directory domain controller accessible from the Rancher server
- An AD service account with read access to the directory
- Network connectivity between Rancher and the AD server (LDAP port 389 or LDAPS port 636)
- AD domain details: base DN, bind DN, and domain name

## Step 1: Prepare the Active Directory Service Account

Create a dedicated service account in AD for Rancher:

1. Open **Active Directory Users and Computers** on your domain controller.
2. Create a new user account:

```plaintext
Username: svc-rancher
Password: <strong-password>
Description: Rancher LDAP service account
Password Never Expires: Yes
User Cannot Change Password: Yes
```

3. Grant the service account read access to the OUs containing your users and groups.

## Step 2: Gather AD Connection Details

Collect the following information from your AD environment:

```plaintext
Domain Controller Hostname: dc01.example.com
Port: 636 (LDAPS) or 389 (LDAP)
Base DN: DC=example,DC=com
Bind DN: CN=svc-rancher,OU=Service Accounts,DC=example,DC=com
User Search Base: OU=Users,DC=example,DC=com
Group Search Base: OU=Groups,DC=example,DC=com
Default Login Domain: example
```

Verify connectivity from the Rancher server:

```bash
# Test LDAP connectivity
ldapsearch -x -H ldap://dc01.example.com:389 \
  -D "CN=svc-rancher,OU=Service Accounts,DC=example,DC=com" \
  -w "<password>" \
  -b "DC=example,DC=com" \
  "(sAMAccountName=testuser)"

# Test LDAPS connectivity
ldapsearch -x -H ldaps://dc01.example.com:636 \
  -D "CN=svc-rancher,OU=Service Accounts,DC=example,DC=com" \
  -w "<password>" \
  -b "DC=example,DC=com" \
  "(sAMAccountName=testuser)"
```

## Step 3: Configure AD Authentication in Rancher

Enable Active Directory authentication:

1. Log in to Rancher as an administrator.
2. Navigate to **Users & Authentication** from the hamburger menu.
3. Click **Auth Provider**.
4. Select **Active Directory**.

## Step 4: Enter Connection Settings

Fill in the AD connection details:

```plaintext
Hostname or IP Address: dc01.example.com
Port: 636
TLS: ☑ Enabled
Certificate: (paste the AD CA certificate if using LDAPS)
Service Account Distinguished Name: CN=svc-rancher,OU=Service Accounts,DC=example,DC=com
Service Account Password: <password>
Default Login Domain: example
```

If your AD uses a self-signed certificate:

```bash
# Export the AD CA certificate
openssl s_client -connect dc01.example.com:636 -showcerts </dev/null 2>/dev/null | \
  openssl x509 -outform PEM > ad-ca-cert.pem
```

Paste the contents of `ad-ca-cert.pem` into the Certificate field.

## Step 5: Configure User and Group Search Settings

Set up the search parameters:

```plaintext
User Search Base: OU=Users,DC=example,DC=com
Username Attribute: sAMAccountName
User Login Attribute: sAMAccountName
User Object Class: person
User Name Attribute: name
User Member Attribute: memberOf
Search Attribute: sAMAccountName|sn|givenName
User Enabled Attribute: userAccountControl
User Disabled Bit Mask: 2

Group Search Base: OU=Groups,DC=example,DC=com
Group Object Class: group
Group Name Attribute: name
Group DN Attribute: distinguishedName
Group Member User Attribute: distinguishedName
Group Member Mapping Attribute: member
Group Search Attribute: sAMAccountName
```

## Step 6: Test the Configuration

Before saving, test the connection:

1. Click **Test** at the bottom of the configuration form.
2. Enter a known AD username and password.
3. Verify that authentication succeeds and user details are returned correctly.

If the test fails, check the following:

- Network connectivity to the AD server.
- Service account credentials are correct.
- The search base DNs point to the correct OUs.
- TLS certificates are properly configured.

## Step 7: Save and Enable

Once testing is successful:

1. Click **Enable** to activate AD authentication.
2. You will be prompted to confirm. Click **Enable** again.

After enabling, the Rancher login page will show an option to log in with Active Directory credentials.

## Step 8: Map AD Groups to Rancher Roles

Assign Rancher roles to AD groups:

1. Navigate to **Users & Authentication** then **Groups**.
2. Search for an AD group.
3. Assign global roles to the group.

```plaintext
AD Group: DevOps-Team
Rancher Role: Standard User

AD Group: Platform-Admins
Rancher Role: Administrator

AD Group: Developers
Rancher Role: User-Base
```

For cluster-level access:

1. Navigate to a cluster.
2. Go to **Cluster Members**.
3. Click **Add**.
4. Search for the AD group.
5. Assign the cluster role.

```plaintext
AD Group: App-Team-A
Cluster Role: Cluster Member

AD Group: SRE-Team
Cluster Role: Cluster Owner
```

## Step 9: Configure Nested Group Support

Enable nested group resolution if your AD uses nested groups:

1. In the AD authentication configuration, look for **Nested Group Membership**.
2. Enable it if your groups contain other groups.

Note that enabling nested group search can increase authentication latency because Rancher must recursively resolve group memberships. Only enable this if your AD structure requires it.

## Step 10: Troubleshoot AD Integration

Common issues and their solutions:

```bash
# Check Rancher server logs for AD errors
kubectl logs -l app=rancher -n cattle-system --tail=200 | grep -i "activedirectory\|ldap\|auth"

# Test LDAP search from the Rancher pod
kubectl exec -it $(kubectl get pod -l app=rancher -n cattle-system -o jsonpath='{.items[0].metadata.name}') \
  -n cattle-system -- \
  ldapsearch -x -H ldaps://dc01.example.com:636 \
  -D "CN=svc-rancher,OU=Service Accounts,DC=example,DC=com" \
  -w "<password>" \
  -b "OU=Users,DC=example,DC=com" \
  "(sAMAccountName=testuser)" dn
```

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Connection timeout | Firewall blocking LDAP ports | Open ports 389/636 between Rancher and AD |
| Invalid credentials | Wrong bind DN format | Use the full distinguished name |
| No users found | Incorrect search base | Verify the OU path in AD |
| TLS handshake failure | Certificate mismatch | Import the correct CA certificate |
| Slow authentication | Nested group resolution | Disable nested groups or optimize AD structure |

## Best Practices

- **Always use LDAPS**: Enable TLS to encrypt authentication traffic between Rancher and Active Directory.
- **Use a dedicated service account**: Do not use a personal account for the LDAP bind. Use a service account with minimal permissions.
- **Map groups, not users**: Assign roles to AD groups rather than individual users for easier management.
- **Test in staging**: Configure and test AD integration in a staging Rancher instance before enabling it in production.
- **Plan for failover**: Configure multiple AD domain controllers for high availability.

## Conclusion

Integrating Active Directory with Rancher brings centralized identity management to your Kubernetes infrastructure. Users can leverage their existing corporate credentials, and administrators can manage access through familiar AD groups. By following the steps in this guide and adhering to security best practices, you can establish a secure and maintainable authentication setup for your Rancher environment.
