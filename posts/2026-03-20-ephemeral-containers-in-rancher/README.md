# How to Use Ephemeral Containers in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Ephemeral Containers, Kubernetes, Debugging, Distroless, Troubleshooting

Description: Use Kubernetes ephemeral containers to debug running pods in Rancher without restarting them, enabling live debugging of distroless and minimal container images.

## Introduction

Ephemeral containers are temporary containers added to a running pod for debugging purposes. Unlike regular containers, they cannot be restarted and don't appear in pod specs after the debugging session ends. They are specifically designed for debugging minimal production images that lack shells and diagnostic tools.

## Prerequisites

- Kubernetes 1.23+ (ephemeral containers are GA since 1.23)
- Rancher with a compatible cluster

## Basic Usage

```bash
# Add an ephemeral debug container to a running pod
kubectl debug -it pod/myapp-7d4f9b6c-xkv8p \
  -n production \
  --image=busybox:latest \
  --target=myapp    # Share process namespace with the 'myapp' container

# Inside the ephemeral container, inspect the main process
ps aux    # Shows processes from the 'myapp' container

# Access the main container filesystem via /proc
ls /proc/1/root/etc/    # View the main container's /etc directory
```

## Debugging Distroless Images

Distroless images contain only the application binary and its dependencies—no shell, no package manager. Ephemeral containers let you attach debug tools:

```bash
# Attach a full debug toolset to a distroless Go application
kubectl debug -it pod/api-server-abc123 \
  -n production \
  --image=gcr.io/distroless/base:debug \    # Distroless debug image has busybox
  --target=api-server

# Or use a more feature-rich debug image
kubectl debug -it pod/api-server-abc123 \
  -n production \
  --image=ubuntu:22.04 \
  --target=api-server
```

## Investigating Memory Issues

```bash
# Attach a debug container with memory analysis tools
kubectl debug -it pod/myapp-abc123 \
  -n production \
  --image=nicolaka/netshoot \
  --target=myapp

# Inside the ephemeral container

# Check memory maps of the main process
cat /proc/1/smaps | grep -E "Size|Rss|Pss" | awk '{sum += $2} END {print sum " kB"}'

# Check heap usage for Java processes
jcmd 1 GC.heap_info    # If main process is Java
```

## Investigating Network Issues

```bash
# Add a network-focused debug container
kubectl debug -it pod/service-abc123 \
  -n production \
  --image=nicolaka/netshoot

# Capture network traffic
tcpdump -i any -n port 8080 -w /tmp/capture.pcap &
sleep 30 && kill %1

# Copy capture to local machine
kubectl cp production/service-abc123:/tmp/capture.pcap ./capture.pcap
```

## Viewing Ephemeral Container Status

```bash
# List ephemeral containers on a pod
kubectl get pod myapp-7d4f9b6c-xkv8p \
  -n production \
  -o jsonpath='{.spec.ephemeralContainers[*].name}'

# Check ephemeral container logs
kubectl logs myapp-7d4f9b6c-xkv8p \
  -n production \
  -c debugger-xxx    # Name assigned to the ephemeral container
```

## Automating Debug Session Setup

```bash
#!/bin/bash
# debug-pod.sh - Quick debug session with standard tools
POD_NAME=$1
NAMESPACE=${2:-default}

kubectl debug -it "pod/${POD_NAME}" \
  --namespace="${NAMESPACE}" \
  --image=nicolaka/netshoot:latest \
  --target="${POD_NAME%%-*}"    # Target container (strips pod suffix)
```

## Conclusion

Ephemeral containers in Rancher fill the gap left by minimal production images. The `--target` flag enables full access to the main container's process namespace, filesystem, and network—all without restarting the pod or modifying the production image. This is the safest way to debug production issues in real time.
