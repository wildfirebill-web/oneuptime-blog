# How to View Pod Processes with podman pod top

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Pods, Monitoring, Processes

Description: Learn how to use podman pod top to view running processes across all containers in a pod.

---

> The podman pod top command shows processes from all containers in a pod in a single unified view.

When debugging a pod, you need to see what processes are running across all containers. The `podman pod top` command gives you a ps-like view of every process in every container within the pod, making it easy to spot runaway processes, zombie processes, or unexpected workloads.

---

## Viewing Processes in a Pod

```bash
# Create a pod with containers
podman pod create --name app-pod
podman run -d --pod app-pod --name web docker.io/library/nginx:alpine
podman run -d --pod app-pod --name worker docker.io/library/alpine \
  sh -c "while true; do sleep 10; done"

# View all processes across the pod
podman pod top app-pod
```

## Understanding the Output

```bash
# Default output shows USER, PID, PPID, and COMMAND
podman pod top app-pod

# Example output:
# USER    PID   PPID  %CPU  COMMAND
# root    1     0     0.0   nginx: master process
# nginx   29    1     0.0   nginx: worker process
# root    1     0     0.0   sleep 10
```

## Custom Format Descriptors

```bash
# Show specific columns
podman pod top app-pod pid user comm vsz rss

# Show process hierarchy with arguments
podman pod top app-pod pid ppid args

# Include CPU and memory usage
podman pod top app-pod pid user pcpu pmem comm
```

## Available Format Descriptors

```bash
# Common descriptors for podman pod top:
# pid    - Process ID
# ppid   - Parent process ID
# user   - Username
# comm   - Command name
# args   - Full command with arguments
# pcpu   - CPU percentage
# pmem   - Memory percentage
# vsz    - Virtual memory size
# rss    - Resident set size
# etime  - Elapsed time
# state  - Process state
```

## Comparing with Individual Container Top

```bash
# View processes in a specific container
podman top web

# View processes in another container
podman top worker

# Pod top combines both into a single view
podman pod top app-pod
```

## Watching Processes Over Time

```bash
# Use watch to continuously monitor pod processes
watch -n 2 podman pod top app-pod pid user pcpu pmem comm

# This refreshes every 2 seconds
```

## Identifying Resource-Heavy Processes

```bash
# Sort output by CPU usage (using shell tools)
podman pod top app-pod pid user pcpu pmem comm | head -1
podman pod top app-pod pid user pcpu pmem comm | tail -n +2 | sort -k3 -rn
```

## Summary

Use `podman pod top` to get a unified view of all processes running across every container in a pod. Customize the output with format descriptors to show CPU, memory, and other process details. Combine with `watch` for continuous monitoring during debugging.
