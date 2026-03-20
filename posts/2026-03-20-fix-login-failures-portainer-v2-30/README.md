# How to Fix Login Failures in Portainer v2.30.0

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Authentication, Login, v2.30, JWT, Upgrade

Description: Learn how to fix login failures introduced in Portainer v2.30.0, including JWT changes, admin password reset procedures, and database migration issues.

---

Portainer v2.30.0 introduced changes to the JWT token format and session handling. Users upgrading from earlier versions may find themselves locked out or unable to log in with correct credentials.

## Step 1: Clear Browser State

Before anything else, clear all browser-stored Portainer state:

```bash
# In browser DevTools (F12) > Application > Storage > Clear site data
# This removes stored JWTs, cookies, and cached login state
```

## Step 2: Check for CSRF / Origin Validation Changes

v2.30.0 tightened origin validation. If you access Portainer via IP and the `--http-disabled` or `--ssl` flags changed, you may see login rejections:

```bash
# Check Portainer logs for origin errors
docker logs portainer 2>&1 | grep -i "origin\|csrf\|referer\|forbidden"
```

If origin errors appear, set the `--base-url` flag to match how you access Portainer:

```bash
docker run ... portainer/portainer-ce:latest --base-url /portainer
```

## Step 3: Reset the Admin Password

If you cannot log in with the correct password:

```bash
# Stop Portainer
docker stop portainer

# Reset the admin password using the reset utility
docker run --rm -v portainer_data:/data portainer/helper-reset-password:latest

# The output will show a new random password
# Use it to log in, then change it in the UI
docker start portainer
```

## Step 4: Check for Database Migration Errors

```bash
# Look for migration errors in the startup logs
docker logs portainer 2>&1 | grep -i "migration\|migrate\|database"

# If there are migration failures, try rolling back one version
docker stop portainer && docker rm portainer

# Run the previous version against the existing data
docker run -d --name portainer \
  -v portainer_data:/data \
  portainer/portainer-ce:2.29.3
```

## Step 5: Verify JWT Secret Integrity

v2.30.0 changed how the JWT signing key is stored. If the key was regenerated during upgrade:

```bash
# All existing sessions are invalid when the JWT key changes
# Simply log in again with your credentials
# If credentials are rejected, reset the admin password (Step 3)
```

## Step 6: Check LDAP/OAuth Configuration

If using external authentication, v2.30.0 changed some LDAP query formats:

1. Go to **Settings > Authentication**.
2. Re-enter and re-test your LDAP/OAuth configuration.
3. Check for changes in the v2.30.0 release notes for your auth provider.
