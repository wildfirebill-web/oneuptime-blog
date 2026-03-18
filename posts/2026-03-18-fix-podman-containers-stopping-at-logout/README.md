# How to Fix Podman Containers Stopping at Logout

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, Systemd, Linux, DevOps

Description: Learn why rootless Podman containers stop when you log out and how to fix it using loginctl enable-linger, systemd user services, and proper session management.

---

> Rootless Podman containers stop when you log out of your SSH session because systemd terminates all user processes. This guide shows you how to keep containers running permanently using lingering sessions and systemd integration.

You SSH into a server, start a Podman container in detached mode, log out, and come back later to find the container has stopped. This is one of the most confusing Podman behaviors for people coming from Docker, where containers keep running regardless of user sessions.

The difference is architectural. Docker uses a system-level daemon that persists independently of user sessions. Podman is daemonless and runs containers as regular user processes. When your user session ends, systemd cleans up all processes belonging to that session, including your containers.

---

## Why Containers Stop at Logout

On modern Linux systems with systemd, user processes are managed by `systemd-logind`. When a user logs out, `systemd-logind` sends SIGTERM (and eventually SIGKILL) to all processes in that user's session scope. This includes any Podman containers running under that user.

You can verify this behavior:

```bash
# Check your current session scope
cat /proc/self/cgroup

# Check if lingering is enabled for your user
loginctl show-user $(whoami) | grep Linger
```

If `Linger=no`, your processes will be killed at logout.

## Fix 1: Enable Lingering for Your User

The primary fix is to enable "lingering" for your user account. Lingering tells systemd to keep a user's processes running even after all their sessions have ended:

```bash
sudo loginctl enable-linger $(whoami)
```

Or for a specific user:

```bash
sudo loginctl enable-linger myuser
```

Verify it is enabled:

```bash
loginctl show-user $(whoami) | grep Linger
```

The output should show `Linger=yes`. After enabling linger, your rootless Podman containers will continue running after you log out.

This is the simplest fix and works immediately for all containers you start from the command line.

## Fix 2: Generate and Use Systemd Unit Files

For production deployments, you should manage containers with systemd rather than running them manually. Podman can generate systemd unit files automatically:

```bash
# First, create and start the container
podman run -d --name my-web-app -p 8080:80 nginx:latest

# Generate a systemd unit file
podman generate systemd --new --name my-web-app --files
```

This creates a file called `container-my-web-app.service`. Move it to your systemd user directory:

```bash
mkdir -p ~/.config/systemd/user/
mv container-my-web-app.service ~/.config/systemd/user/

# Reload systemd
systemctl --user daemon-reload

# Enable and start the service
systemctl --user enable --now container-my-web-app.service
```

The `--new` flag is important. It generates a unit file that creates a fresh container each time the service starts, rather than trying to start an existing stopped container. This is more reliable.

Check the service status:

```bash
systemctl --user status container-my-web-app.service
```

## Fix 3: Use Podman Quadlet (Recommended for Newer Systems)

On systems with Podman 4.4 or later, Quadlet is the recommended way to manage containers with systemd. Instead of generating unit files, you write a simple `.container` file:

Create `~/.config/containers/systemd/my-web-app.container`:

```ini
[Unit]
Description=My Web Application

[Container]
Image=nginx:latest
PublishPort=8080:80
ContainerName=my-web-app

[Service]
Restart=always

[Install]
WantedBy=default.target
```

Then reload and start:

```bash
systemctl --user daemon-reload
systemctl --user start my-web-app.service
systemctl --user enable my-web-app.service
```

Quadlet files are simpler to write and maintain than full systemd unit files. They support all common Podman options:

```ini
[Container]
Image=my-app:latest
ContainerName=my-app
PublishPort=8080:80
Volume=/data:/app/data:z
Environment=NODE_ENV=production
Environment=PORT=80
```

## Fix 4: Run Containers as Root (System-Level)

If you run Podman as root, containers are managed by the system-level cgroup and are not tied to user sessions:

```bash
sudo podman run -d --name my-app nginx:latest
```

For system-level systemd management:

```bash
sudo podman run -d --name my-app nginx:latest
sudo podman generate systemd --new --name my-app --files
sudo mv container-my-app.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now container-my-app.service
```

The downside is that rootful containers run with full root privileges, which is less secure than rootless Podman.

## Fix 5: Use tmux or screen as a Workaround

As a temporary workaround, you can run containers inside a tmux or screen session. These terminal multiplexers keep their processes alive because they run as a separate session:

```bash
# Start a tmux session
tmux new-session -d -s containers

# Run your container inside the tmux session
tmux send-keys -t containers "podman run -d --name my-app nginx:latest" Enter
```

However, this is a workaround, not a solution. The tmux process itself can be killed if lingering is not enabled and the system is aggressive about cleanup. Use lingering or systemd for production.

## Fix 6: Ensure XDG_RUNTIME_DIR Persists

Podman relies on `XDG_RUNTIME_DIR` (typically `/run/user/<UID>`) for runtime data. When you log out without lingering, this directory is removed, which kills any Podman processes that depend on it.

Enabling linger (Fix 1) also preserves `XDG_RUNTIME_DIR`. You can verify it exists:

```bash
echo $XDG_RUNTIME_DIR
ls -la $XDG_RUNTIME_DIR
```

If you see errors about `XDG_RUNTIME_DIR` not being set or not existing when you connect via `su` or `sudo su`, set it manually:

```bash
export XDG_RUNTIME_DIR=/run/user/$(id -u)
```

## Complete Setup for Production

Here is a complete workflow for setting up a rootless Podman container that survives logout and starts on boot:

```bash
# Step 1: Enable lingering
sudo loginctl enable-linger $(whoami)

# Step 2: Create the Quadlet file
mkdir -p ~/.config/containers/systemd/

cat > ~/.config/containers/systemd/my-app.container << 'EOF'
[Unit]
Description=My Application
After=network-online.target

[Container]
Image=my-app:latest
ContainerName=my-app
PublishPort=8080:80
Volume=app-data:/app/data
Environment=NODE_ENV=production
AutoUpdate=registry

[Service]
Restart=always
TimeoutStartSec=120

[Install]
WantedBy=default.target
EOF

# Step 3: Reload and enable
systemctl --user daemon-reload
systemctl --user enable --now my-app.service

# Step 4: Verify
systemctl --user status my-app.service
podman ps
```

## Verifying Containers Survive Logout

To test that your containers survive logout:

```bash
# Start your container
podman run -d --name test-survive nginx:latest

# Check it is running
podman ps

# Log out
exit

# Log back in (new SSH session)
podman ps
```

If the container is still listed as running, your fix is working.

## Troubleshooting

If containers still stop after enabling linger:

```bash
# Verify linger is actually enabled
ls /var/lib/systemd/linger/
# Your username should be listed

# Check systemd user instance is running
systemctl --user status

# Check for errors in the journal
journalctl --user -xe

# Make sure your systemd user instance starts at boot
sudo loginctl enable-linger $(whoami)
```

## Conclusion

Rootless Podman containers stop at logout because systemd terminates user session processes. The fix is straightforward: run `loginctl enable-linger` to keep your user's processes alive, then use systemd unit files or Quadlet to manage containers properly. For production, Quadlet files provide the cleanest integration with systemd, handling container lifecycle, restart policies, and boot startup automatically. Always enable lingering for any user account that runs long-lived Podman containers.
