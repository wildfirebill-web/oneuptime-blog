# How to Create a Pod with IPC Namespace Sharing in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Pods, IPC, Namespaces

Description: Learn how to use IPC namespace sharing in Podman pods for shared memory communication between containers.

---

> IPC namespace sharing lets containers in a pod use shared memory segments, semaphores, and message queues.

Inter-Process Communication (IPC) namespace sharing enables containers to exchange data through POSIX shared memory, System V semaphores, and message queues. This is essential for applications that rely on shared memory for high-performance data transfer, such as database engines communicating with proxy processes.

---

## Default IPC Sharing in Pods

```bash
# IPC is shared by default in Podman pods
podman pod create --name ipc-pod

# Verify IPC is in the shared namespaces
podman pod inspect ipc-pod --format '{{.SharedNamespaces}}'
# Output includes: ipc
```

## Creating Shared Memory Between Containers

```bash
# Create a pod with IPC sharing
podman pod create --name shm-pod --share ipc,net

# Run a producer container that writes to shared memory
podman run -d --pod shm-pod --name producer docker.io/library/alpine \
  sh -c "echo 'shared data from producer' > /dev/shm/mydata && sleep 3600"

# Run a consumer container that reads from shared memory
podman run --pod shm-pod --name consumer docker.io/library/alpine \
  cat /dev/shm/mydata
# Output: shared data from producer
```

## Configuring Shared Memory Size

```bash
# Set the shared memory size for the pod
podman pod create --name large-shm-pod --shm-size 1g --share ipc,net

# Verify the shared memory size inside a container
podman run --rm --pod large-shm-pod docker.io/library/alpine df -h /dev/shm
# Shows 1.0G available
```

## Use Case: Database with Connection Pooler

```bash
# PostgreSQL and PgBouncer can communicate via Unix sockets in shared memory
podman pod create --name db-pod --share ipc,net -p 5432:5432

# Run PostgreSQL
podman run -d --pod db-pod --name postgres \
  -e POSTGRES_PASSWORD=secret \
  docker.io/library/postgres:16-alpine

# Run a sidecar that uses shared IPC
podman run -d --pod db-pod --name sidecar docker.io/library/alpine sleep 3600

# Both can access /dev/shm
podman exec postgres ls /dev/shm
podman exec sidecar ls /dev/shm
```

## Disabling IPC Sharing

```bash
# Create a pod without IPC sharing
podman pod create --name no-ipc-pod --share net

# Containers have separate IPC namespaces
podman run -d --pod no-ipc-pod --name a docker.io/library/alpine \
  sh -c "echo 'data' > /dev/shm/test && sleep 3600"

podman run --pod no-ipc-pod --name b docker.io/library/alpine \
  cat /dev/shm/test
# File not found - IPC is not shared
```

## Verifying IPC Isolation

```bash
# Check IPC namespace IDs for containers in the pod
podman exec producer cat /proc/1/ns/ipc
podman exec consumer cat /proc/1/ns/ipc
# With IPC sharing, both show the same namespace ID
```

## Summary

IPC namespace sharing is enabled by default in Podman pods and allows containers to communicate through shared memory, semaphores, and message queues. Use `--shm-size` to configure the shared memory pool size. Disable IPC sharing by omitting `ipc` from the `--share` flag if containers should not access each other's IPC resources.
