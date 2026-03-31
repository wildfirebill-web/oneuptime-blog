# How to Use loginctl enable-linger with Rootless Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Rootless, Systemd, Linger, loginctl

Description: A detailed guide to using loginctl enable-linger to keep rootless Podman containers running after user logout, with real-world examples and troubleshooting tips.

---

> "Linger is the bridge between a user session experiment and a production-grade rootless deployment."

The `loginctl enable-linger` command is essential for any serious rootless Podman deployment. Without it, systemd kills all your user processes -- including containers -- the moment your last session ends. This guide explains what linger does, how to configure it, and how to verify it works correctly with Podman.

---

## What loginctl enable-linger Does

By default, systemd manages user services through a user manager instance that starts when you log in and stops when you log out. Enabling linger changes this behavior:

```bash
# Without linger:

# Login  -> systemd starts user@1000.service
# Logout -> systemd stops user@1000.service (kills all user processes)

# With linger:
# Boot   -> systemd starts user@1000.service
# Login  -> user@1000.service already running
# Logout -> user@1000.service keeps running
# Reboot -> user@1000.service starts at boot

# Check if linger is currently enabled for your user
loginctl show-user $USER --property=Linger
# Output: Linger=no (default)
```

## Enabling Linger

Enable linger for your user account:

```bash
# Enable linger for the current user
loginctl enable-linger

# Or enable it for a specific user (requires root)
sudo loginctl enable-linger username

# Verify it is enabled
loginctl show-user $USER --property=Linger
# Output: Linger=yes

# Linger creates a file in this directory
ls /var/lib/systemd/linger/
# You should see a file named after your username
```

## Verifying the User Manager Persists

After enabling linger, confirm that your user manager stays running after logout:

```bash
# Check your user manager status
systemctl --user status

# See your user's systemd slice
systemctl status user-$(id -u).slice

# List all user services currently running
systemctl --user list-units --type=service --state=running

# Even after logout, this process should exist
ps aux | grep "systemd --user"
```

## Creating a Rootless Container Service

With linger enabled, create a systemd user service for your container:

```bash
# Run the container first to test it
podman run -d --name myservice -p 9090:80 nginx

# Generate the systemd unit file
podman generate systemd --name myservice --new --files
# Creates: container-myservice.service

# Install the unit file
mkdir -p ~/.config/systemd/user
mv container-myservice.service ~/.config/systemd/user/

# Remove the test container
podman stop myservice && podman rm myservice

# Reload systemd and enable the service
systemctl --user daemon-reload
systemctl --user enable --now container-myservice.service
```

## Testing the Full Workflow

Run through the complete test to confirm containers survive logout:

```bash
# Step 1: Verify linger is on
loginctl show-user $USER --property=Linger

# Step 2: Verify the service is running
systemctl --user is-active container-myservice.service

# Step 3: Verify the container is running
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Step 4: Open a second SSH session, then close the first
# In the second session, check the container
podman ps

# Step 5: Close ALL sessions, wait, then log back in
# The container should still be running
podman ps
systemctl --user status container-myservice.service
```

## Troubleshooting Linger Issues

Common issues and how to resolve them:

```bash
# Issue: Linger enabled but containers still stop after logout
# Check if the service is truly a user service
systemctl --user status container-myservice.service

# Make sure you are not using system-level units by mistake
# User units go in ~/.config/systemd/user/
# System units go in /etc/systemd/system/ (wrong for rootless)

# Issue: "Failed to enable linger" error
# Check if your account is allowed to enable linger
# Some systems restrict this via polkit
cat /etc/polkit-1/rules.d/*linger* 2>/dev/null

# Issue: User manager not starting at boot despite linger
# Check the user manager status
systemctl status user-$(id -u).slice
journalctl --user --boot -p err

# Issue: XDG_RUNTIME_DIR not set after SSH login
# Verify the environment variable is set
echo $XDG_RUNTIME_DIR
# Should output: /run/user/1000 (or your UID)

# If missing, ensure pam_systemd is enabled
grep pam_systemd /etc/pam.d/sshd
```

## Summary

`loginctl enable-linger` is a one-time setup step that transforms rootless Podman from a development tool into a production-ready container runtime. It tells systemd to start your user manager at boot and keep it running regardless of login sessions. Enable it with `loginctl enable-linger`, create systemd user services with `podman generate systemd`, and your containers will persist across logouts and reboots. Always verify the setup by testing a full logout cycle before relying on it in production.
