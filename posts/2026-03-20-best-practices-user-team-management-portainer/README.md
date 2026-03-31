# Best Practices for User and Team Management in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, RBAC, User Management, Teams, Security, Access Control, Best Practice

Description: Establish secure and scalable user and team management in Portainer with RBAC, SSO integration, least-privilege access policies, and user lifecycle management.

---

Proper user and team management in Portainer prevents unauthorized access, ensures accountability, and makes operations auditable. These best practices apply to both small teams and large organizations.

## Principle of Least Privilege

Grant users only the access they need:

| Role | Who Gets It | What They Can Do |
|------|-------------|-----------------|
| Administrator | Platform engineers only | Full access everywhere |
| Environment Admin | Team leads | Full access to their environment |
| Helpdesk | Support staff | Read-only + logs/console |
| Standard User | Developers | Deploy in their namespace |
| Read-Only | Auditors, stakeholders | View only |

Never grant Administrator to regular developers.

## Team Structure

Create teams that mirror your organizational structure:

```text
Teams:
- platform-engineers    → Environment Admin on all environments
- backend-team          → Standard User on backend environments
- frontend-team         → Standard User on frontend environments
- ml-team               → Standard User on ML environments
- security-auditors     → Read-Only on all environments
```

## LDAP/SSO Integration

For organizations with Active Directory or LDAP, configure SSO to avoid manual user management:

In Portainer BE, go to **Settings > Authentication > LDAP**:

```text
Server: ldap.company.com:389
Reader DN: cn=portainer-reader,ou=service-accounts,dc=company,dc=com
Base DN: dc=company,dc=com
Username attribute: sAMAccountName
Group base DN: ou=groups,dc=company,dc=com
Group membership: member
Auto-create users: true

Group mappings:
  CN=Container-Admins  → Administrator
  CN=Container-Devs    → Standard User
  CN=Container-Auditors → Read-Only User
```

With LDAP sync, users are automatically added and removed as they join/leave AD groups.

## User Lifecycle Management

**Onboarding:**
1. Add user to the appropriate LDAP/AD group (or create manually in Portainer)
2. Assign to the correct team in Portainer
3. Verify access is scoped to only their environments

**Offboarding:**
1. Remove user from LDAP groups (or disable/delete in Portainer)
2. Review any active sessions
3. Rotate any shared credentials the user had access to

## Audit Logging (BE Feature)

Enable audit logging to track all user actions:

- User logins and logouts
- Container start/stop/delete operations
- Stack deployments and updates
- Registry configuration changes

Review audit logs regularly, especially after incidents.

## Service Accounts for Automation

For CI/CD pipelines and automation scripts, use dedicated service accounts:

1. Create a dedicated user: `ci-deploy-service`
2. Assign only the permissions needed (e.g., stack deployment on specific environments)
3. Generate an API token for this user (not your personal account)
4. Store the token in your secrets manager (Vault, AWS Secrets Manager, etc.)

```bash
# Create service account via Portainer API

curl -X POST "https://portainer.example.com/api/users" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"username": "ci-deploy", "password": "generated-strong-password", "role": 2}'

# Generate API token for service account
curl -X POST "https://portainer.example.com/api/auth" \
  -d '{"username": "ci-deploy", "password": "generated-strong-password"}'
```

## Regular Access Reviews

Schedule quarterly access reviews:
- List all users and their roles
- Verify each user still needs access
- Check for orphaned service accounts
- Remove or downgrade access for users who have changed roles

## Summary

User and team management in Portainer requires deliberate structure. Use LDAP/SSO to eliminate manual user management, apply least-privilege roles, use service accounts for automation, and review access regularly. These practices reduce the blast radius of compromised accounts and prevent accidental access to production environments.
