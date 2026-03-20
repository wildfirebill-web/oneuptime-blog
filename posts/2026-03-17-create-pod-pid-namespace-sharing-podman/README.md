# How to Create a Pod with PID Namespace Sharing in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Pods, PID, Namespaces

Description: Learn how to enable PID namespace sharing in Podman pods so containers can see each other's processes.

---

> PID namespace sharing lets containers in a pod see and signal each other's processes.

By default, each container in a Podman pod has its own PID namespace. Enabling PID sharing merges the process trees so every container can see processes from all other containers in the pod. This is useful for sidecar patterns where a monitoring agent needs to observe application processes, or where containers need to send signals to each other.

---

## Enabling PID Namespace Sharing

```bash
# Create a pod with PID namespace sharing

podman pod create --name pid-pod --share pid,net,ipc

# Run an application container
podman run -d --pod pid-pod --name app docker.io/library/alpine \
  sh -c "while true; do echo working; sleep 5; done"

# Run a monitoring sidecar
podman run -d --pod pid-pod --name monitor docker.io/library/alpine sleep 3600
```

## Viewing Processes Across Containers

```bash
# From the monitor container, see all processes including app's
podman exec monitor ps aux

# Output shows processes from both containers:
# PID   USER   COMMAND
# 1     root   /pause (infra container)
# 7     root   sh -c while true; do echo working; sleep 5; done
# 15    root   sleep 3600
# 18    root   sleep 5
```

## Sending Signals Between Containers

```bash
# From the monitor container, send a signal to the app process
# First find the PID of the app's main process
APP_PID=$(podman exec monitor ps aux | grep "echo working" | awk '{print $1}' | head -1)

# Send SIGUSR1 to the app process
podman exec monitor kill -USR1 "$APP_PID"
```

## Use Case: Process Monitoring Sidecar

```bash
# Create a pod with PID sharing for observability
podman pod create --name monitored-app --share pid,net -p 8080:80

# Run the main application
podman run -d --pod monitored-app --name web docker.io/library/nginx:alpine

# Run a sidecar that monitors nginx processes
podman run -d --pod monitored-app --name watcher docker.io/library/alpine \
  sh -c "while true; do
    NGINX_PROCS=\$(ps aux | grep -c '[n]ginx')
    echo \"nginx processes: \$NGINX_PROCS\"
    sleep 10
  done"

# View the watcher output
podman logs -f watcher
```

## Comparing With and Without PID Sharing

```bash
# Without PID sharing (default)
podman pod create --name no-pid --share net
podman run -d --pod no-pid --name a1 docker.io/library/alpine sleep 3600
podman run -d --pod no-pid --name a2 docker.io/library/alpine sleep 3600
podman exec a2 ps aux
# Only sees its own processes

# With PID sharing
podman pod create --name with-pid --share pid,net
podman run -d --pod with-pid --name b1 docker.io/library/alpine sleep 3600
podman run -d --pod with-pid --name b2 docker.io/library/alpine sleep 3600
podman exec b2 ps aux
# Sees processes from both b1 and b2
```

## Summary

Enable PID namespace sharing with `--share pid` when creating a pod to allow containers to see and signal each other's processes. This is essential for monitoring sidecars, process managers, and debugging tools that need visibility into other containers' process trees.
