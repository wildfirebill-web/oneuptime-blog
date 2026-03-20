# How to Use a Non-Default Admin Username in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Admin, Authentication, Hardening

Description: Learn how to configure a non-default admin username in Portainer to reduce the attack surface against brute-force login attempts.

## Why Change the Default Admin Username?

The default Portainer admin username `admin` is widely known. Automated attack tools will target this username first. Using a non-obvious username adds security through obscurity as a first line of defense.

## Setting a Custom Admin Username During Initial Setup

Portainer allows you to set a custom admin username only during the initial setup (either via the UI wizard or the API):

### Via the Portainer UI

On first launch:
1. Navigate to `https://your-portainer-host:9443`.
2. In the initial setup form, change the **Username** field from `admin` to your chosen username.
3. Set a strong password.
4. Click **Create user**.

### Via the API

```bash
# Create initial admin with custom username

curl -X POST "https://portainer.mycompany.com/api/users/admin/init" \
  -H "Content-Type: application/json" \
  -d '{
    "Username": "portainer-ops",
    "Password": "Str0ngP@ssword!2024"
  }'
```

## Creating an Admin User with a Custom Name After Setup

If Portainer is already initialized with `admin`, create an additional admin and rename/delete the old one:

```bash
# Step 1: Create a new admin user with your preferred username
TOKEN=$(curl -sf -X POST "${PORTAINER_URL}/api/auth" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"currentpassword"}' | jq -r '.jwt')

curl -X POST "${PORTAINER_URL}/api/users" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Username": "ops-admin",
    "Password": "NewStr0ngP@ss!",
    "Role": 1
  }'
```

Then log in with the new admin account and delete the old `admin` account:

```bash
# Step 2: Log in as the new admin
NEW_TOKEN=$(curl -sf -X POST "${PORTAINER_URL}/api/auth" \
  -H "Content-Type: application/json" \
  -d '{"username":"ops-admin","password":"NewStr0ngP@ss!"}' | jq -r '.jwt')

# Step 3: Get the ID of the old admin user
OLD_ADMIN_ID=$(curl -s "${PORTAINER_URL}/api/users" \
  -H "Authorization: Bearer ${NEW_TOKEN}" | \
  jq '.[] | select(.Username == "admin") | .Id')

# Step 4: Delete the old admin account
curl -X DELETE "${PORTAINER_URL}/api/users/${OLD_ADMIN_ID}" \
  -H "Authorization: Bearer ${NEW_TOKEN}"
```

## Using LDAP for Username Management

For larger organizations, consider LDAP authentication (available in Portainer BE):

1. Go to **Settings > Authentication**.
2. Select **LDAP**.
3. Configure your LDAP server.
4. Users authenticate with their corporate directory credentials.

This centralizes user management and removes the need for local Portainer accounts entirely.

## Other Username Security Tips

- Use an email address format (`ops@company.com`) to avoid dictionary attacks.
- Combine with a strong password policy (minimum 16 characters).
- Enable login failure lockout where available.
- Rotate admin passwords quarterly.

## Conclusion

Using a non-default admin username is a simple security improvement that costs nothing and makes brute-force attacks significantly harder. Always set a custom username during initial Portainer setup.
