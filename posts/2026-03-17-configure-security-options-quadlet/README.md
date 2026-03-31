# How to Configure Security Options in Quadlet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Quadlet, Security, SELinux, Seccomp

Description: Learn how to configure security options in Podman Quadlet container files including SELinux, seccomp profiles, and Linux capabilities.

---

> Harden your containerized applications by configuring security options such as SELinux labels, seccomp profiles, and capability controls in Quadlet container files.

Security is a critical concern when running containers. Quadlet lets you configure security options declaratively, including SELinux contexts, seccomp profiles, dropped capabilities, and read-only filesystem settings.

---

## Dropping All Capabilities

Run a container with minimal Linux capabilities:

```ini
# ~/.config/containers/systemd/secure-app.container

[Unit]
Description=Secure application with minimal capabilities

[Container]
Image=docker.io/myorg/myapp:latest
PublishPort=8080:8080

# Drop all capabilities and add back only what is needed
DropCapability=all
AddCapability=net_bind_service

[Service]
Restart=on-failure

[Install]
WantedBy=default.target
```

## Read-Only Root Filesystem

Prevent the container from writing to its root filesystem:

```ini
# ~/.config/containers/systemd/readonly-app.container
[Unit]
Description=Application with read-only root filesystem

[Container]
Image=docker.io/myorg/myapp:latest
ReadOnly=true
# Provide a writable tmpfs for temp files
Tmpfs=/tmp:size=50M
Tmpfs=/run:size=10M
Volume=appdata.volume:/app/data

[Service]
Restart=on-failure

[Install]
WantedBy=default.target
```

## Configuring SELinux Labels

```ini
# ~/.config/containers/systemd/selinux-app.container
[Unit]
Description=Application with custom SELinux context

[Container]
Image=docker.io/myorg/myapp:latest
SecurityLabelType=container_t
SecurityLabelLevel=s0:c100,c200

[Service]
Restart=on-failure

[Install]
WantedBy=default.target
```

## Disabling SELinux Labeling

```ini
[Container]
Image=docker.io/myorg/myapp:latest
# Disable SELinux label separation
SecurityLabelDisable=true
```

## Custom Seccomp Profile

```ini
[Container]
Image=docker.io/myorg/myapp:latest
# Use a custom seccomp profile
PodmanArgs=--security-opt=seccomp=/path/to/custom-seccomp.json
```

## No New Privileges

Prevent the container process from gaining additional privileges:

```ini
[Container]
Image=docker.io/myorg/myapp:latest
PodmanArgs=--security-opt=no-new-privileges:true
DropCapability=all
ReadOnly=true
```

## Verify Security Settings

```bash
# Reload and start
systemctl --user daemon-reload
systemctl --user start secure-app.service

# Check capabilities
podman inspect secure-app --format '{{.HostConfig.CapDrop}}'
podman inspect secure-app --format '{{.HostConfig.CapAdd}}'

# Check if read-only
podman inspect secure-app --format '{{.HostConfig.ReadonlyRootfs}}'
```

## Summary

Quadlet provides native directives like `DropCapability`, `AddCapability`, `ReadOnly`, and SELinux label options, plus `PodmanArgs` for additional security options. Drop unnecessary capabilities, use read-only root filesystems, configure SELinux labels, apply custom seccomp profiles, and prevent privilege escalation. Following the principle of least privilege helps protect your host and other containers.
