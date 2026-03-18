# How to Configure systemd to Restart Podman Containers on Failure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, systemd, Restart, Reliability

Description: Learn how to configure systemd restart policies for Podman containers to automatically recover from crashes and failures.

---

> Configure systemd to automatically restart failed Podman containers with proper delays, retry limits, and failure handling for production reliability.

When a containerized application crashes, you want it to restart automatically. Systemd provides flexible restart policies that control when and how often a service restarts after failure.

---

## Basic Restart Configuration

In a Quadlet file, add restart settings to the `[Service]` section:

```ini
# ~/.config/containers/systemd/myapp.container
[Unit]
Description=Application with restart policy

[Container]
Image=docker.io/myorg/myapp:latest
PublishPort=3000:3000

[Service]
# Restart only when the container exits with a non-zero status
Restart=on-failure
# Wait 5 seconds before restarting
RestartSec=5

[Install]
WantedBy=default.target
```

## Restart Policy Options

| Policy | Restarts on | Use case |
|--------|------------|----------|
| `no` | Never | One-shot tasks |
| `on-failure` | Non-zero exit | Most services |
| `on-abnormal` | Signal, timeout, watchdog | Signal-sensitive apps |
| `on-success` | Clean exit (code 0) | Periodic jobs |
| `always` | Any exit | Critical services |

## Rate Limiting Restarts

Prevent infinite restart loops:

```ini
[Unit]
Description=Application with restart limits
# Allow 5 restart attempts in 300 seconds
StartLimitIntervalSec=300
StartLimitBurst=5

[Container]
Image=docker.io/myorg/myapp:latest
PublishPort=3000:3000

[Service]
Restart=on-failure
RestartSec=10
```

If the service restarts 5 times within 300 seconds, systemd stops trying and marks the service as failed.

## Exponential Backoff with RestartSteps

For newer systemd versions, configure exponential backoff:

```ini
[Service]
Restart=on-failure
RestartSec=1
RestartMaxDelaySec=60
RestartSteps=10
```

This starts with a 1-second delay and increases up to 60 seconds over 10 steps.

## Running a Command on Failure

Execute a notification or cleanup command when the service fails:

```ini
[Service]
Restart=on-failure
RestartSec=5
# Run a script when the service fails permanently
ExecStopPost=/bin/sh -c 'if [ "$$EXIT_STATUS" != "0" ]; then echo "Service failed" >> /tmp/failures.log; fi'
```

## Monitoring Restart Behavior

```bash
# Check restart count
systemctl --user show myapp.service --property=NRestarts

# View restart-related journal entries
journalctl --user -u myapp.service | grep -i restart

# Check if the service hit its restart limit
systemctl --user status myapp.service
```

## Reset a Failed Service

When a service hits its restart limit:

```bash
# Reset the failure counter
systemctl --user reset-failed myapp.service

# Start the service again
systemctl --user start myapp.service
```

## Summary

Systemd restart policies provide production-grade reliability for Podman containers. Use `Restart=on-failure` for most services, set `RestartSec` for a delay between attempts, and configure `StartLimitIntervalSec` and `StartLimitBurst` to prevent infinite restart loops. Monitor restarts with `systemctl show` and reset failed services with `systemctl reset-failed`.
