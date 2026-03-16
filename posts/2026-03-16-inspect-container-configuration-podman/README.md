# How to Inspect a Container's Configuration in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Container Inspection, Configuration

Description: Learn how to use podman inspect to view detailed configuration information about containers, including networking, volumes, environment variables, and runtime settings.

---

> podman inspect reveals every detail about a container's configuration, from network settings to runtime parameters.

Understanding a container's full configuration is essential for debugging, auditing, and documentation. The `podman inspect` command returns a comprehensive JSON document containing every detail about a container. This guide shows you how to use it to extract the information you need.

---

## Basic Inspection

The simplest form returns the full JSON configuration:

```bash
# Start a container for testing
podman run -d --name my-app -p 8080:80 -e APP_ENV=production nginx:latest

# Inspect the container (returns full JSON)
podman inspect my-app
```

This outputs a large JSON document. Let us look at how to extract specific fields.

## Common Inspection Queries

### Container State and Status

```bash
# Get the container's current state
podman inspect my-app --format '{{.State.Status}}'
# Output: running

# Get the start time
podman inspect my-app --format '{{.State.StartedAt}}'

# Check if the container is running
podman inspect my-app --format '{{.State.Running}}'
# Output: true

# Get the process ID on the host
podman inspect my-app --format '{{.State.Pid}}'
```

### Network Configuration

```bash
# Get the IP address
podman inspect my-app --format '{{.NetworkSettings.IPAddress}}'

# Get all network settings
podman inspect my-app --format '{{json .NetworkSettings}}' | python3 -m json.tool

# Get port mappings
podman inspect my-app --format '{{json .NetworkSettings.Ports}}' | python3 -m json.tool

# Get the gateway
podman inspect my-app --format '{{.NetworkSettings.Gateway}}'
```

### Environment Variables

```bash
# List all environment variables
podman inspect my-app --format '{{json .Config.Env}}' | python3 -m json.tool

# Get environment as a readable list
podman inspect my-app --format '{{range .Config.Env}}{{println .}}{{end}}'
```

### Image and Command Information

```bash
# Get the image used
podman inspect my-app --format '{{.Config.Image}}'

# Get the command being run
podman inspect my-app --format '{{json .Config.Cmd}}'

# Get the entrypoint
podman inspect my-app --format '{{json .Config.Entrypoint}}'

# Get the working directory
podman inspect my-app --format '{{.Config.WorkingDir}}'
```

### Volume and Mount Information

```bash
# Start a container with volumes for testing
podman run -d --name vol-app -v /tmp/data:/data:z -v my-vol:/app/data nginx:latest

# Get mount information
podman inspect vol-app --format '{{json .Mounts}}' | python3 -m json.tool

# List mount points
podman inspect vol-app --format '{{range .Mounts}}{{.Source}} -> {{.Destination}} ({{.Type}}){{println}}{{end}}'
```

### Resource Limits

```bash
# Start a container with resource limits
podman run -d --name limited-app --memory 256m --cpus 0.5 nginx:latest

# Check memory limit
podman inspect limited-app --format '{{.HostConfig.Memory}}'

# Check CPU configuration
podman inspect limited-app --format 'CPU Period: {{.HostConfig.CpuPeriod}}, CPU Quota: {{.HostConfig.CpuQuota}}'
```

### Container Metadata

```bash
# Get the container ID
podman inspect my-app --format '{{.Id}}'

# Get the container name
podman inspect my-app --format '{{.Name}}'

# Get the creation time
podman inspect my-app --format '{{.Created}}'

# Get the restart policy
podman inspect my-app --format '{{json .HostConfig.RestartPolicy}}'
```

## Inspecting Multiple Containers

You can inspect multiple containers at once:

```bash
# Inspect several containers
podman inspect my-app vol-app limited-app --format '{{.Name}}: {{.State.Status}}'
```

## Inspecting Images

`podman inspect` also works on images:

```bash
# Inspect an image
podman inspect --type image nginx:latest --format '{{json .Config.ExposedPorts}}'

# Get the image size
podman inspect --type image nginx:latest --format '{{.Size}}'

# Get the image layers
podman inspect --type image nginx:latest --format '{{json .RootFS.Layers}}' | python3 -m json.tool
```

## Saving Inspection Output

Save the full configuration for documentation or comparison:

```bash
# Save full inspection to a file
podman inspect my-app > /tmp/container-config.json

# Pretty-print and save
podman inspect my-app | python3 -m json.tool > /tmp/container-config-pretty.json

# Compare two containers
diff <(podman inspect my-app | python3 -m json.tool) <(podman inspect limited-app | python3 -m json.tool)
```

## Practical Debugging Workflow

Use inspect as part of a debugging workflow:

```bash
# Quick container health summary
podman inspect my-app --format '
Name: {{.Name}}
Status: {{.State.Status}}
Image: {{.Config.Image}}
IP: {{.NetworkSettings.IPAddress}}
Ports: {{json .NetworkSettings.Ports}}
Started: {{.State.StartedAt}}
PID: {{.State.Pid}}'
```

## Cleanup

```bash
podman stop my-app vol-app limited-app 2>/dev/null
podman rm my-app vol-app limited-app 2>/dev/null
podman volume rm my-vol 2>/dev/null
```

## Summary

The `podman inspect` command is your primary tool for examining container configuration. Use `--format` with Go templates to extract specific fields, pipe to `python3 -m json.tool` for readable JSON output, and combine it with other commands for comprehensive debugging workflows. It works on both containers and images.
