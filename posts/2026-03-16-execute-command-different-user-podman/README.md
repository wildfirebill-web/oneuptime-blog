# How to Execute a Command as a Different User in a Podman Container

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Container Exec, User Management

Description: Learn how to run commands as different users inside Podman containers using the --user flag, including root, named users, and UID/GID combinations.

---

> Running commands as the right user inside a container is critical for proper permissions, security testing, and application debugging.

Podman containers often run processes as a specific user defined in the Dockerfile. However, there are times when you need to execute commands as a different user, whether to gain root access for debugging or to test application behavior under a specific user context. This guide covers all the ways to switch users when executing commands in Podman containers.

---

## The --user Flag

The `--user` flag on `podman exec` lets you override the default user for the executed command:

```bash
# Start a container for testing

podman run -d --name my-app nginx:latest

# Execute a command as root
podman exec --user root my-app whoami
# Output: root

# Execute a command as the www-data user
podman exec --user www-data my-app whoami
# Output: www-data
```

## Specifying Users by Name

You can use any username that exists in the container's `/etc/passwd` file:

```bash
# Check which users exist in the container
podman exec my-app cat /etc/passwd

# Run a command as the nginx user
podman exec --user nginx my-app id
# Output: uid=101(nginx) gid=101(nginx) groups=101(nginx)

# Run a command as nobody
podman exec --user nobody my-app id
```

## Specifying Users by UID

You can also specify users by their numeric user ID:

```bash
# Run as UID 0 (root)
podman exec --user 0 my-app whoami
# Output: root

# Run as UID 1000
podman exec --user 1000 my-app id
# Output: uid=1000 gid=0(root)

# Run as UID 101 (nginx)
podman exec --user 101 my-app whoami
```

## Specifying User and Group Together

You can specify both UID and GID in the format `user:group`:

```bash
# Run as root user with root group
podman exec --user root:root my-app id

# Run as UID 1000 with GID 1000
podman exec --user 1000:1000 my-app id
# Output: uid=1000 gid=1000

# Run as nginx user with www-data group
podman exec --user nginx:www-data my-app id

# Mix names and numbers
podman exec --user root:1000 my-app id
```

## Practical Use Cases

### Debugging Permission Issues

When your application fails due to permission problems, run commands as the application user to understand the issue:

```bash
# Check what the default container user can access
podman exec my-app ls -la /var/log/nginx/

# Try accessing a file as a different user to test permissions
podman exec --user www-data my-app cat /var/log/nginx/error.log

# Check file ownership
podman exec --user root my-app find /var/log/nginx -type f -exec ls -la {} \;
```

### Installing Packages as Root

Many containers run as non-root users by default. To install debugging tools, switch to root:

```bash
# This might fail if the container runs as non-root
podman exec my-app apt-get update
# Error: Permission denied

# Switch to root to install packages
podman exec --user root my-app apt-get update
podman exec --user root my-app apt-get install -y curl vim strace
```

### Testing Application Security

Verify that your application properly restricts access for different users:

```bash
# Create a test file as root
podman exec --user root my-app touch /tmp/secure-file
podman exec --user root my-app chmod 600 /tmp/secure-file

# Verify other users cannot read it
podman exec --user nobody my-app cat /tmp/secure-file
# Expected: Permission denied

# Verify root can read it
podman exec --user root my-app cat /tmp/secure-file
```

### Running Database Commands

Database containers often require specific users:

```bash
# Start a PostgreSQL container
podman run -d --name my-db -e POSTGRES_PASSWORD=secret postgres:latest

# Run psql as the postgres user
podman exec --user postgres my-db psql -c "SELECT version();"

# Check database files as postgres user
podman exec --user postgres my-db ls -la /var/lib/postgresql/data/
```

## Combining --user with Interactive Shells

Open an interactive shell as a specific user:

```bash
# Root shell for full admin access
podman exec -it --user root my-app /bin/bash

# Shell as application user for debugging
podman exec -it --user nginx my-app /bin/sh

# Shell as a specific UID/GID
podman exec -it --user 1000:1000 my-app /bin/bash
```

## Checking the Default User

Before switching users, it helps to know what the default user is:

```bash
# Check the container's configured user
podman inspect my-app --format '{{.Config.User}}'

# Check who the current process runs as
podman exec my-app whoami

# Get full user details
podman exec my-app id
```

## Handling Non-Existent Users

If you specify a UID that does not exist in the container, the command still runs but without a named user:

```bash
# Run as a UID with no /etc/passwd entry
podman exec --user 9999 my-app id
# Output: uid=9999 gid=0(root)

# whoami will show "I have no name!" or similar
podman exec --user 9999 my-app whoami
# Output: whoami: cannot find name for user ID 9999
```

This is useful for testing how your application handles unknown user contexts.

## Cleanup

```bash
podman stop my-app my-db 2>/dev/null
podman rm my-app my-db 2>/dev/null
```

## Summary

The `--user` flag on `podman exec` provides flexible user switching for commands inside containers. Use username strings, numeric UIDs, or `user:group` combinations to execute commands with the exact permissions you need. This is essential for debugging permission issues, installing tools, and testing application security boundaries.
