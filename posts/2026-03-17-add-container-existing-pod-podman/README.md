# How to Add a Container to an Existing Pod in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Pod, Orchestration

Description: Learn how to add new containers to an already running Podman pod.

---

> You can add containers to an existing pod at any time using the --pod flag with podman run.

Pods in Podman are not static. You can add containers to a running pod to introduce new services, attach debugging tools, or scale up a component. Each new container joins the pod's shared network namespace and can immediately communicate with other containers over localhost.

---

## Creating a Pod and Adding the First Container

```bash
# Create a pod

podman pod create --name app-pod -p 8080:80

# Add the first container
podman run -d --pod app-pod --name web docker.io/library/nginx:alpine
```

## Adding a Second Container to the Running Pod

```bash
# Add a sidecar container to the same pod
podman run -d --pod app-pod --name log-agent docker.io/library/alpine \
  sh -c "while true; do wget -qO- http://localhost:80 >> /tmp/access.log; sleep 10; done"

# Both containers share the same network namespace
podman exec log-agent cat /tmp/access.log
```

## Adding a Debugging Container

```bash
# Attach a debug container with networking tools
podman run -it --pod app-pod --name debug docker.io/library/alpine sh

# Inside the debug container, you can inspect the pod network
# ip addr show
# wget -qO- http://localhost:80
# netstat -tlnp
```

## Verifying the New Container Is in the Pod

```bash
# List all containers in the pod
podman ps --filter pod=app-pod --format "{{.Names}} {{.Status}}"

# Inspect the pod to see container count
podman pod inspect app-pod --format '{{.NumContainers}}'
```

## Adding a Container with Environment Variables

```bash
# Add a database container with configuration
podman run -d --pod app-pod --name db \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=myapp \
  docker.io/library/postgres:16-alpine

# The web container can now connect to postgres on localhost:5432
podman exec web sh -c "apk add --no-cache postgresql-client && pg_isready -h localhost"
```

## Adding a Container with a Volume

```bash
# Add a container that mounts a volume for shared data
podman run -d --pod app-pod --name exporter \
  -v /tmp/metrics:/data \
  docker.io/library/alpine \
  sh -c "while true; do date >> /data/heartbeat.txt; sleep 30; done"
```

## Removing a Single Container from the Pod

```bash
# Stop and remove just one container without affecting the pod
podman stop log-agent
podman rm log-agent

# The pod and other containers continue running
podman pod inspect app-pod --format '{{.NumContainers}}'
```

## Summary

Adding containers to an existing Podman pod is as simple as using `--pod <pod-name>` with `podman run`. New containers immediately share the pod's network namespace and can communicate with other pod members over localhost. You can add, remove, and replace individual containers without disturbing the rest of the pod.
