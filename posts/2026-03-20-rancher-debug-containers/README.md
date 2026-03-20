# How to Set Up Debug Containers in Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Debug Containers, Troubleshooting, Development

Description: Use debug containers in Rancher to troubleshoot running pods without modifying production images, using kubectl debug and custom sidecar approaches.

## Introduction

Debug containers allow you to troubleshoot running pods by injecting a container with debugging tools into an existing pod, without modifying or rebuilding the production image. Kubernetes 1.23+ introduced stable support for ephemeral containers via `kubectl debug`. This guide covers multiple approaches to debug containers in Rancher-managed clusters.

## Prerequisites

- Rancher-managed cluster with Kubernetes 1.23+
- kubectl with debug capabilities
- Appropriate RBAC permissions for ephemeral containers

## Step 1: Enable Ephemeral Containers (Pre-1.25)

```bash
# Ephemeral containers are enabled by default in Kubernetes 1.25+

# For earlier versions, check the feature gate
kubectl get --raw /version | jq .gitVersion

# Verify the feature is enabled
kubectl auth can-i create pods/ephemeralcontainers
```

## Step 2: Debug a Running Pod

```bash
# Attach a debug container to a running pod
kubectl debug -it \
  -n production \
  pod/my-app-pod-xyz \
  --image=busybox \
  --target=my-app

# Use a more feature-rich debug image
kubectl debug -it \
  -n production \
  pod/my-app-pod-xyz \
  --image=nicolaka/netshoot \
  --target=my-app

# The --target flag shares the process namespace with the target container
# so you can see its processes and filesystem
```

## Step 3: Create a Debug Image

```dockerfile
# Dockerfile.debug - Comprehensive debug image
FROM alpine:3.18

RUN apk add --no-cache \
    bash \
    curl \
    wget \
    tcpdump \
    netcat-openbsd \
    nmap \
    bind-tools \
    strace \
    ltrace \
    gdb \
    jq \
    vim \
    procps \
    lsof \
    htop \
    iperf3 \
    openssl \
    postgresql-client \
    mysql-client \
    redis \
    python3 \
    py3-pip \
    && pip3 install httpie

CMD ["/bin/bash"]
```

```bash
# Build and push the debug image
docker build -t registry.example.com/debug-tools:latest -f Dockerfile.debug .
docker push registry.example.com/debug-tools:latest

# Use your custom debug image
kubectl debug -it \
  pod/my-app-pod \
  --image=registry.example.com/debug-tools:latest \
  -n production
```

## Step 4: Debug a Node

```bash
# Debug a node directly
kubectl debug node/worker-node-01 \
  -it \
  --image=registry.example.com/debug-tools:latest

# Access the node filesystem at /host
ls /host/etc/
cat /host/var/log/syslog

# Check node processes
chroot /host ps aux

# Check kubelet logs on the node
chroot /host journalctl -u kubelet -f
```

## Step 5: Copy-and-Debug Pattern

```bash
# Create a copy of the pod with a modified image for debugging
# This creates a new pod with the same spec but an interactive shell
kubectl debug \
  -n production \
  pod/my-app-pod \
  --copy-to=my-app-debug \
  --container=my-app \
  -it \
  --image=registry.example.com/debug-tools:latest \
  -- bash

# Copy pod and override the entrypoint
kubectl debug \
  -n production \
  pod/my-app-pod \
  --copy-to=my-app-debug-shell \
  --set-image=my-app=registry.example.com/app:debug \
  -it \
  -- /bin/sh
```

## Step 6: Sidecar Debug Pattern for Production

```yaml
# deployment-with-debug-sidecar.yaml - Conditional debug sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  annotations:
    debug-mode: "false"
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: registry.example.com/app:v1.0
          ports:
            - containerPort: 8080
        # Debug sidecar (only add when needed)
        # - name: debug-sidecar
        #   image: registry.example.com/debug-tools:latest
        #   command: ["sleep", "infinity"]
        #   securityContext:
        #     capabilities:
        #       add: ["SYS_PTRACE"]
```

## Step 7: Configure RBAC for Debug Access

```yaml
# debug-rbac.yaml - Allow developers to use debug containers
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: debug-pods
  namespace: development
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/ephemeralcontainers"]
    verbs: ["patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers-debug
  namespace: development
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: debug-pods
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
```

## Step 8: Debug Network Issues

```bash
# Use netshoot for network debugging
kubectl debug -it \
  pod/my-app-pod \
  --image=nicolaka/netshoot \
  -n production \
  -- bash

# Inside netshoot container
# Check DNS resolution
dig my-service.production.svc.cluster.local

# Check connectivity
curl -v http://other-service:8080/health

# Capture traffic
tcpdump -i any -w /tmp/capture.pcap port 8080

# Analyze with Wireshark locally
kubectl cp production/netshoot-pod:/tmp/capture.pcap ./capture.pcap
```

## Conclusion

Debug containers in Rancher provide a powerful way to investigate issues in running applications without modifying production images. The `kubectl debug` command is the recommended approach for Kubernetes 1.23+, offering both ephemeral container injection and pod copy patterns. Combined with a well-equipped debug image and appropriate RBAC policies, debug containers enable efficient production troubleshooting while maintaining security boundaries.
