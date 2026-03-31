# How to Enable a Quadlet Service to Start at Boot

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Quadlet, Systemd, Boot, Startup

Description: Learn how to configure Quadlet container services to start automatically at boot using systemd enable and linger.

---

> Ensure your Quadlet container services start automatically when the system boots by enabling them with systemctl and configuring user session persistence with loginctl.

Having containers start automatically at boot is essential for production services. Quadlet integrates with systemd's enable mechanism and, for rootless containers, requires user linger to persist services beyond login sessions.

---

## Enable a Quadlet Service

The `[Install]` section in your Quadlet file defines the boot target:

```ini
# ~/.config/containers/systemd/webapp.container

[Unit]
Description=Web application

[Container]
Image=docker.io/myorg/webapp:latest
PublishPort=8080:80

[Service]
Restart=on-failure

[Install]
# This line makes the service eligible for boot startup
WantedBy=default.target
```

Enable the service:

```bash
# Reload systemd to detect the Quadlet file
systemctl --user daemon-reload

# Enable the service to start at boot
systemctl --user enable webapp.service

# Start it now as well
systemctl --user start webapp.service

# Verify it is enabled
systemctl --user is-enabled webapp.service
```

## Enable User Linger for Rootless Services

Rootless systemd services only run when the user is logged in by default. Enable linger to keep them running:

```bash
# Enable linger for the current user
loginctl enable-linger $(whoami)

# Verify linger is enabled
loginctl show-user $(whoami) --property=Linger
```

Without linger, rootless Quadlet services will stop when you log out and will not start at boot.

## Rootful Services at Boot

For rootful containers, place files in `/etc/containers/systemd/` and use system-level systemctl:

```bash
# Copy the Quadlet file to the system directory
sudo cp webapp.container /etc/containers/systemd/

# Reload systemd
sudo systemctl daemon-reload

# Enable and start
sudo systemctl enable --now webapp.service
```

## Verify Boot Startup

```bash
# Reboot and check
# After reboot, verify the service started
systemctl --user status webapp.service

# Check the boot log for the service
journalctl --user -u webapp.service -b
```

## Enable Multiple Services

```bash
# Enable all your application services
systemctl --user enable app-db.service
systemctl --user enable app-api.service
systemctl --user enable app-proxy.service

# Or start and enable in one command
systemctl --user enable --now webapp.service
```

## Disable Boot Startup

```bash
# Disable the service from starting at boot
systemctl --user disable webapp.service

# The service will no longer start automatically
systemctl --user is-enabled webapp.service
```

## Summary

To have Quadlet services start at boot, include `WantedBy=default.target` in the `[Install]` section and run `systemctl enable`. For rootless containers, enable user linger with `loginctl enable-linger` so services persist beyond login sessions. Rootful services use system-level systemctl and do not require linger.
