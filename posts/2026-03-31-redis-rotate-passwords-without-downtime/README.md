# How to Rotate Redis Passwords Without Downtime

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Security, Password, ACL, Operation

Description: Learn how to rotate Redis passwords without dropping connections or causing downtime by temporarily assigning two passwords to a user during the transition.

---

Rotating Redis passwords is a common security requirement. The naive approach - changing the password and restarting clients - causes downtime. Redis ACL supports multiple passwords per user, making zero-downtime rotation straightforward.

## How Multiple Passwords Work

Redis ACL users can have more than one password. A client authenticated with any valid password for that user remains authorized. This lets you add a new password before removing the old one.

## Step-by-Step Zero-Downtime Rotation

### Step 1: Add the New Password

Keep the old password active and add the new one:

```bash
ACL SETUSER myapp >new-secret-password
```

The user now accepts both the old and new passwords.

### Step 2: Update Application Configuration

Deploy your updated application config with the new password. Since both passwords are valid, clients with the old password still work during the rollout.

### Step 3: Verify New Password Works

```bash
redis-cli -u redis://myapp:new-secret-password@127.0.0.1:6379 PING
# PONG
```

### Step 4: Remove the Old Password

Once all clients are using the new password, remove the old one:

```bash
ACL SETUSER myapp <old-secret-password
```

The `<` prefix removes a specific password. Clients still using the old password will now get authentication errors.

### Step 5: Persist the Changes

```bash
ACL SAVE
```

## Automating Rotation with a Script

```bash
#!/bin/bash
OLD_PASS="$1"
NEW_PASS="$2"
USER="myapp"

# Add new password
redis-cli ACL SETUSER "$USER" ">$NEW_PASS"

echo "New password added. Deploy application changes now."
echo "Press Enter when all clients are using the new password..."
read

# Remove old password
redis-cli ACL SETUSER "$USER" "<$OLD_PASS"

# Persist changes
redis-cli ACL SAVE

echo "Old password removed. Rotation complete."
```

## Rotate the Default User Password

For the built-in `default` user (used when no ACL is set):

```bash
# Add new password, keep old one
ACL SETUSER default >new-requirepass

# After all clients updated, remove old password
ACL SETUSER default <old-requirepass
ACL SAVE
```

## Verify Current Passwords

Check how many passwords a user has (hashed, not plaintext):

```bash
ACL GETUSER myapp
```

```text
3) "passwords"
4) 1) "fa7a..."   <- hashed new password
   2) "c3ab..."   <- hashed old password (still active)
```

## Summary

Redis supports multiple passwords per ACL user, enabling zero-downtime rotation. Add the new password with `>newpass`, wait for all clients to update, then remove the old password with `<oldpass`. Always call `ACL SAVE` to persist changes to disk.
