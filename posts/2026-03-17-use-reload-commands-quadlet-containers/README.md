# How to Use Reload Commands in Quadlet Containers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Quadlet, Systemd, Reload

Description: Learn how to configure reload commands in Quadlet containers to gracefully reload application configuration without restarting the container.

---

> Configure ExecReload in your Quadlet container files to gracefully reload application configuration by sending signals to the container process without a full restart.

Many applications support reloading their configuration on the fly when they receive a specific signal (like SIGHUP for Nginx). Quadlet lets you define a reload command in the `[Service]` section, enabling `systemctl reload` to trigger a configuration refresh.

---

## How Reload Works

When you run `systemctl reload`, systemd executes the `ExecReload` command instead of stopping and restarting the service. For containers, this typically means sending a signal to the main process inside the container.

## Configuring Reload for Nginx

Nginx reloads its configuration when it receives a SIGHUP signal:

```ini
# ~/.config/containers/systemd/nginx.container
[Unit]
Description=Nginx with reload support

[Container]
Image=docker.io/library/nginx:latest
ContainerName=nginx
PublishPort=8080:80
Volume=%h/.config/nginx/nginx.conf:/etc/nginx/nginx.conf:ro,Z

[Service]
Restart=on-failure
# Send SIGHUP to the container's main process to reload config
ExecReload=/usr/bin/podman kill --signal=HUP nginx

[Install]
WantedBy=default.target
```

## Using Reload

```bash
# Start the service
systemctl --user daemon-reload
systemctl --user start nginx.service

# Edit the Nginx configuration
vim ~/.config/nginx/nginx.conf

# Reload without restarting
systemctl --user reload nginx.service

# Verify the reload succeeded
systemctl --user status nginx.service
```

## Configuring Reload for Apache

```ini
# ~/.config/containers/systemd/httpd.container
[Unit]
Description=Apache with reload support

[Container]
Image=docker.io/library/httpd:latest
ContainerName=httpd
PublishPort=8080:80

[Service]
Restart=on-failure
# Apache uses USR1 for graceful reload
ExecReload=/usr/bin/podman kill --signal=USR1 httpd

[Install]
WantedBy=default.target
```

## Configuring Reload for HAProxy

```ini
# ~/.config/containers/systemd/haproxy.container
[Unit]
Description=HAProxy with reload support

[Container]
Image=docker.io/library/haproxy:latest
ContainerName=haproxy
PublishPort=80:80
Volume=%h/.config/haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro,Z

[Service]
Restart=on-failure
# HAProxy uses SIGUSR2 for reload
ExecReload=/usr/bin/podman kill --signal=USR2 haproxy

[Install]
WantedBy=default.target
```

## Reload with Configuration Validation

Run a validation check before reloading:

```ini
[Service]
Restart=on-failure
# Test config before reloading Nginx
ExecReload=/bin/sh -c '/usr/bin/podman exec nginx nginx -t && /usr/bin/podman kill --signal=HUP nginx'
```

This ensures the reload only happens if the configuration is valid.

## Summary

The `ExecReload` directive in Quadlet container files enables `systemctl reload` to gracefully refresh application configuration without restarting the container. Typically this involves sending a signal like SIGHUP to the container's main process using `podman kill --signal`. You can chain a configuration test before the reload to ensure safety.
