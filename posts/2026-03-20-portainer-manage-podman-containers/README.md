# How to Manage Podman Containers from Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Podman, Container Management, Docker-Compatible API, Linux

Description: Learn how to use Portainer to start, stop, inspect, and manage Podman containers through the Docker-compatible API, with notes on Podman-specific behaviors.

---

Once Portainer is connected to a Podman socket, you can manage Podman containers through the familiar Portainer UI. Most container operations work identically to Docker, with a few differences to be aware of.

## Container Operations via Portainer

After connecting Portainer to the Podman socket, the standard container operations work:

| Operation | Portainer UI | Works with Podman? |
|---|---|---|
| Start/Stop/Restart | Containers list | Yes |
| View Logs | Container > Logs tab | Yes |
| Execute command | Container > Console tab | Yes |
| Inspect container | Container > Inspect tab | Yes |
| View stats | Container > Stats tab | Yes (limited) |
| Pull images | Images > Pull | Yes |

## Starting a Container via Portainer

1. Go to **Containers > Add Container**.
2. Set the image (pulled from a registry or local).
3. Configure ports, volumes, and environment variables.
4. Click **Deploy the container**.

Portainer sends a Docker-compatible API call to Podman, which creates and starts the container.

## Viewing Container Logs

```bash
# Equivalent Podman command for what Portainer does:
podman logs <container-name>

# Portainer provides real-time log streaming via the API
# which Podman's socket supports natively
```

## Exec into a Container

In Portainer go to **Containers > [container] > Console** and click **Connect**.

Portainer opens a WebSocket terminal that calls Podman's exec API — identical to `podman exec -it <container> /bin/sh`.

## Known Differences from Docker

**Rootless containers:** Podman running rootless may show different user IDs in Portainer's stats view. The stats API is supported but some metrics (like network stats) may be unavailable without root or extra capabilities.

**Pods:** Podman's pod concept (grouping containers) is not visible in Portainer. Each container in a pod appears as a separate item.

**cgroups v2:** Some Portainer stats features require cgroups v2, which Podman fully supports on modern Linux systems.

## Updating Containers

In Portainer, recreate the container to pull a newer image:

1. Go to the container.
2. Click **Duplicate/Edit**.
3. Check **Always pull the image**.
4. Click **Deploy the container**.

Portainer will pull the updated image via Podman's pull API and recreate the container.
