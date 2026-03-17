# How to Create a Quadlet Image Unit File

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Quadlet, Systemd, Images

Description: Learn how to create a Quadlet .image unit file to manage container image pulling and tagging as a systemd resource.

---

> Quadlet image unit files ensure container images are pulled and available before containers start, with optional auto-update support.

Quadlet `.image` files manage the lifecycle of container images within systemd. They define which image to pull, from which registry, and can be referenced by container unit files to ensure the image is present before the container starts.

---

## Basic Image Unit File

```ini
# ~/.config/containers/systemd/nginx.image
[Image]
Image=docker.io/library/nginx:alpine
```

## Image with Auto-Update

```ini
# ~/.config/containers/systemd/app-image.image
[Image]
Image=docker.io/library/python:3.12-slim
# Podman will check for newer versions of this image
```

## Referencing Images in Containers

```ini
# ~/.config/containers/systemd/nginx.image
[Image]
Image=docker.io/library/nginx:alpine

# ~/.config/containers/systemd/web.container
[Container]
Image=nginx.image
PublishPort=8080:80

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```bash
# Reload and start — the image is pulled first
systemctl --user daemon-reload
systemctl --user start web

# Verify the image is present
podman images | grep nginx
```

## Image from a Private Registry

```ini
# ~/.config/containers/systemd/private-app.image
[Image]
Image=registry.example.com/myteam/myapp:v1.5
```

```bash
# Login to the registry first
podman login registry.example.com

# Then start the service — it pulls from the private registry
systemctl --user start myapp
```

## Multiple Image Files

```ini
# ~/.config/containers/systemd/postgres.image
[Image]
Image=docker.io/library/postgres:16-alpine

# ~/.config/containers/systemd/redis.image
[Image]
Image=docker.io/library/redis:7-alpine

# ~/.config/containers/systemd/node.image
[Image]
Image=docker.io/library/node:20-alpine
```

## Using Images with Containers

```ini
# ~/.config/containers/systemd/postgres.image
[Image]
Image=docker.io/library/postgres:16-alpine

# ~/.config/containers/systemd/db.container
[Container]
Image=postgres.image
Environment=POSTGRES_PASSWORD=secret
Volume=pgdata.volume:/var/lib/postgresql/data
PublishPort=5432:5432

[Service]
Restart=always

[Install]
WantedBy=default.target
```

## Auto-Updating Images

```bash
# Enable Podman auto-update timer
systemctl --user enable --now podman-auto-update.timer

# Check auto-update status
podman auto-update --dry-run

# The timer periodically checks for new image versions
# and restarts containers using updated images
```

## Image with TLS Verification

```ini
# ~/.config/containers/systemd/insecure-app.image
[Image]
Image=insecure-registry.local:5000/myapp:latest
# For testing with self-signed certificates
PodmanArgs=--tls-verify=false
```

## Managing Images

```bash
# Check which images are managed by Quadlet
podman images

# Force pull a new version
podman pull docker.io/library/nginx:alpine

# Restart the container to use the new image
systemctl --user restart web
```

## Summary

Quadlet `.image` files manage container image pulling as systemd resources. Reference them in `.container` files with `Image=name.image` to ensure images are present before containers start. Combine with Podman auto-update for automatic image refreshes when new versions are published.
