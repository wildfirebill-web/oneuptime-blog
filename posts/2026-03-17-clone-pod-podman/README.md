# How to Clone a Pod with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Pods, Cloning

Description: Learn how to clone a Podman pod to create a duplicate with the same configuration.

---

> Cloning a pod lets you replicate its configuration for testing, scaling, or creating parallel environments.

Podman provides the `podman pod clone` command to create a copy of an existing pod with its configuration. The cloned pod gets the same network settings, port mappings, and namespace sharing as the original. This is useful for creating staging copies, running parallel tests, or scaling workloads.

---

## Cloning a Pod

```bash
# Create a source pod

podman pod create --name original-pod -p 8080:80
podman run -d --pod original-pod --name web docker.io/library/nginx:alpine

# Clone the pod
podman pod clone original-pod

# The cloned pod gets an auto-generated name
podman pod ls
```

## Cloning with a Custom Name

```bash
# Clone and assign a specific name
podman pod clone --name cloned-pod original-pod

# Verify the clone
podman pod ls --format "table {{.Name}}\t{{.Status}}\t{{.NumContainers}}"
```

## Cloning with Different Port Mappings

```bash
# Clone but override the port mapping
podman pod clone --name staging-pod -p 9090:80 original-pod

# The staging pod listens on port 9090 instead of 8080
podman pod inspect staging-pod --format '{{.InfraConfig.PortBindings}}'
```

## Adding Containers to the Cloned Pod

```bash
# The cloned pod starts empty (infra container only)
# Add containers to it
podman run -d --pod cloned-pod --name web-clone docker.io/library/nginx:alpine
podman run -d --pod cloned-pod --name cache-clone docker.io/library/redis:7-alpine

# Verify containers in the clone
podman ps --filter pod=cloned-pod
```

## Cloning for A/B Testing

```bash
# Create version A
podman pod create --name app-v1 -p 8081:80
podman run -d --pod app-v1 --name web-v1 docker.io/library/nginx:1.24-alpine

# Clone for version B with a different port
podman pod clone --name app-v2 -p 8082:80 app-v1
podman run -d --pod app-v2 --name web-v2 docker.io/library/nginx:1.25-alpine

# Both pods run simultaneously on different ports
curl http://localhost:8081
curl http://localhost:8082
```

## Comparing Original and Clone

```bash
# Inspect both pods to compare configuration
diff <(podman pod inspect original-pod | jq '.InfraConfig') \
     <(podman pod inspect cloned-pod | jq '.InfraConfig')
```

## Summary

Use `podman pod clone` to create a copy of a pod with the same configuration. Assign a custom name with `--name` and override port mappings as needed. Cloned pods start with only the infra container, so you need to add your application containers to the new pod. This is valuable for creating parallel environments, A/B testing, and staging deployments.
