# How to View Pod Logs in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Pods, Logging, Debugging

Description: Learn how to view and follow logs from containers within a Podman pod.

---

> Viewing pod logs lets you see the output of all containers in a pod to diagnose issues and monitor behavior.

Podman provides `podman pod logs` to aggregate logs from all containers in a pod. You can also view logs for individual containers within the pod. This guide covers both approaches.

---

## Viewing Logs from a Specific Container in a Pod

```bash
# Create a pod with containers
podman pod create --name app-pod -p 8080:80
podman run -d --pod app-pod --name web docker.io/library/nginx:alpine
podman run -d --pod app-pod --name app docker.io/library/alpine \
  sh -c "while true; do echo 'app heartbeat'; sleep 5; done"

# View logs from the web container
podman logs web

# View logs from the app container
podman logs app
```

## Following Logs in Real Time

```bash
# Follow logs from a container (like tail -f)
podman logs -f web

# Follow with timestamps
podman logs -f --timestamps web
```

## Viewing Recent Logs

```bash
# Show only the last 20 lines
podman logs --tail 20 web

# Show logs since a specific time
podman logs --since 2m app
```

## Viewing Logs from All Containers in a Pod

```bash
# Use podman pod logs to see all container output
podman pod logs app-pod

# Follow all pod logs
podman pod logs -f app-pod

# Show container names in the output
podman pod logs --names app-pod
```

## Combining Logs with Timestamps

```bash
# View pod logs with timestamps for chronological ordering
podman pod logs --timestamps --names app-pod

# Show only recent logs
podman pod logs --tail 50 --names app-pod
```

## Filtering Logs by Container

```bash
# View logs from a specific container within the pod context
podman pod logs --container web app-pod

# Combine with tail and follow
podman pod logs --container web --tail 10 -f app-pod
```

## Saving Logs to a File

```bash
# Save all pod logs to a file for later analysis
podman pod logs --names --timestamps app-pod > /tmp/pod-logs.txt

# Save logs from a specific container
podman logs web > /tmp/web-access.log 2>&1
```

## Summary

View pod logs with `podman pod logs` to see output from all containers, or use `podman logs <container>` for individual containers. Use `--names` to identify which container produced each line, `--timestamps` for chronological ordering, and `-f` to follow logs in real time.
