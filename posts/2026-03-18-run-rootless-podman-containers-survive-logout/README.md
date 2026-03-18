# How to Run Rootless Podman Containers That Survive Logout

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Rootless, systemd, Linger, Persistence

Description: A practical guide to keeping rootless Podman containers running after you log out of your session, using systemd user services and loginctl linger.

---

> "A container that stops when you close your terminal is not ready for production."

By default, rootless Podman containers are tied to your user session. When you log out, systemd terminates your user slice and all processes in it, including your containers. This guide shows you how to make rootless containers persistent, surviving logouts and even reboots.

---

## Understanding Why Containers Stop

When you log out, systemd kills all processes in your user session. Rootless Podman containers run under your user scope, so they get terminated too.

```bash
# Start a rootless container
podman run -d --name test-app nginx

# Check which systemd scope it runs under
podman inspect test-app --format '{{.State.ConmonPid}}' | xargs ps -o user,pid,cgroup -p

# Log out and log back in -- the container will be gone
podman ps -a --filter name=test-app
# The container will show as stopped or missing
```

## Step 1: Enable Linger for Your User

The `loginctl enable-linger` command tells systemd to keep your user manager running even when you have no active sessions.

```bash
# Enable linger for your user
loginctl enable-linger $USER

# Verify linger is enabled
loginctl show-user $USER --property=Linger
# Output: Linger=yes

# Check that your user manager is running
systemctl --user status
```

With linger enabled, systemd starts your user instance at boot and keeps it running permanently.

## Step 2: Generate a systemd User Service

Podman can automatically generate systemd unit files from running containers.

```bash
# Start the container you want to persist
podman run -d --name webapp -p 8080:80 nginx

# Generate a systemd user service file
podman generate systemd --name webapp --new --files

# This creates a file named container-webapp.service
cat container-webapp.service
```

The `--new` flag generates a service that creates and removes the container each time, which is cleaner than managing an existing container.

## Step 3: Install the Service

Move the generated service file to your systemd user directory and enable it:

```bash
# Create the systemd user directory if it does not exist
mkdir -p ~/.config/systemd/user

# Move the generated service file
mv container-webapp.service ~/.config/systemd/user/

# Stop and remove the manually started container
podman stop webapp
podman rm webapp

# Reload the systemd user daemon
systemctl --user daemon-reload

# Enable the service to start at boot
systemctl --user enable container-webapp.service

# Start the service now
systemctl --user start container-webapp.service

# Check the service status
systemctl --user status container-webapp.service
```

## Step 4: Verify Persistence Across Logout

Test that the container survives a logout:

```bash
# Verify the container is running
podman ps

# Check the service is active
systemctl --user is-active container-webapp.service

# Log out via SSH and log back in, then check again
# The container should still be running
podman ps
systemctl --user status container-webapp.service
```

## Managing the Persistent Container

Use standard systemctl commands to manage the container lifecycle:

```bash
# Stop the container
systemctl --user stop container-webapp.service

# Start the container
systemctl --user start container-webapp.service

# Restart the container
systemctl --user restart container-webapp.service

# View container logs through journald
journalctl --user -u container-webapp.service --no-pager -n 50

# Follow logs in real time
journalctl --user -u container-webapp.service -f
```

## Handling Multiple Containers with Pods

For multi-container applications, generate a service for an entire pod:

```bash
# Create a pod with multiple containers
podman pod create --name myapp -p 8080:80 -p 5432:5432

# Add containers to the pod
podman run -d --pod myapp --name myapp-web nginx
podman run -d --pod myapp --name myapp-db postgres:16

# Generate systemd files for the entire pod
podman generate systemd --name myapp --new --files

# Install all the generated service files
mv pod-myapp.service container-myapp-*.service ~/.config/systemd/user/

# Remove the manually created pod
podman pod rm -f myapp

# Enable and start the pod service
systemctl --user daemon-reload
systemctl --user enable --now pod-myapp.service

# Verify the pod is running
podman pod ps
podman ps --pod
```

## Summary

Making rootless Podman containers survive logout requires two things: enabling linger with `loginctl enable-linger` and managing containers through systemd user services. Use `podman generate systemd` to create service files, install them under `~/.config/systemd/user/`, and enable them with `systemctl --user enable`. This gives you containers that start at boot, survive logouts, restart on failure, and integrate with standard systemd management workflows.
