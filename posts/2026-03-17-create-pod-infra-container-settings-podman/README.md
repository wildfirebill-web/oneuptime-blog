# How to Create a Pod with Infra Container Settings in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Pods, Infra Container, Configuration

Description: Learn how to customize the infra container in a Podman pod for specific networking and namespace requirements.

---

> The infra container is the foundation of every pod, holding shared namespaces alive even when all other containers stop.

Every Podman pod has an infra container that runs a minimal pause process. This container owns the shared namespaces (network, IPC, UTS) and keeps them alive for the lifetime of the pod. You can customize the infra container's image, command, and resource limits.

---

## Understanding the Infra Container

```bash
# Create a pod and observe the infra container

podman pod create --name my-pod

# List all containers including the infra container
podman ps -a --filter pod=my-pod --format "table {{.Names}}\t{{.Image}}\t{{.Command}}"

# The infra container runs the k8s.gcr.io/pause image or a local equivalent
```

## Using a Custom Infra Image

```bash
# Specify a custom infra container image
podman pod create --name custom-infra-pod \
  --infra-image docker.io/library/alpine:latest

# The infra container uses the specified image
podman ps -a --filter pod=custom-infra-pod --format "{{.Names}} {{.Image}}"
```

## Setting a Custom Infra Command

```bash
# Override the infra container's command
podman pod create --name cmd-pod \
  --infra-command "/bin/sleep inf"

# The infra container runs sleep instead of the default pause
```

## Disabling the Infra Container

```bash
# Create a pod without an infra container
podman pod create --name no-infra-pod --infra=false

# Without an infra container, namespaces are not preserved
# when containers stop and restart
podman pod ls --filter name=no-infra-pod
```

## Configuring Infra Container Networking

```bash
# Port mappings are applied to the infra container
podman pod create --name web-pod \
  -p 8080:80 \
  -p 8443:443

# The infra container owns these port bindings
INFRA_ID=$(podman pod inspect web-pod --format '{{.InfraContainerId}}')
podman inspect "$INFRA_ID" --format '{{.HostConfig.PortBindings}}'
```

## Setting Resource Limits on the Infra Container

```bash
# The infra container is lightweight but you can set limits
podman pod create --name limited-pod \
  --infra-conmon-pidfile /tmp/pod-conmon.pid

# Inspect infra container resource usage
podman pod stats limited-pod --no-stream
```

## Inspecting the Infra Container

```bash
# Get the infra container ID
podman pod inspect my-pod --format '{{.InfraContainerId}}'

# Inspect the infra container in detail
INFRA_ID=$(podman pod inspect my-pod --format '{{.InfraContainerId}}')
podman inspect "$INFRA_ID" | jq '{
  Image: .Config.Image,
  Cmd: .Config.Cmd,
  NetworkMode: .HostConfig.NetworkMode,
  Namespaces: .HostConfig.NamespaceOptions
}'
```

## Summary

The infra container is the backbone of every Podman pod, maintaining shared namespaces for all member containers. Customize it with `--infra-image` for a different base image, `--infra-command` for a different entry process, or `--infra=false` to disable it entirely. Port mappings and namespace configuration all flow through the infra container.
