# How to Use Kubewarden Policy Hub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, Policy, PolicyHub, Security

Description: Learn how to discover, evaluate, and deploy pre-built policies from the Kubewarden Policy Hub to accelerate your cluster security implementation.

## Introduction

The Kubewarden Policy Hub (https://hub.kubewarden.io) is a community registry of pre-built, ready-to-use admission policies. Instead of writing policies from scratch, you can search the hub for policies that address your specific needs — from pod security to resource quotas, network policies, and more — and deploy them directly to your cluster.

This guide covers how to discover policies on the hub, evaluate them with `kwctl`, and deploy them to your Kubernetes cluster.

## Prerequisites

- Kubewarden installed on your cluster
- `kwctl` CLI installed
- `kubectl` access to your cluster

## Accessing the Policy Hub

### Via Web Browser

Visit https://hub.kubewarden.io to browse policies:
- Search by keyword (e.g., "privileged", "resources", "image")
- Filter by category (Security, Best Practices, Custom)
- View policy documentation and settings

### Via kwctl CLI

```bash
# Search for policies in the hub
kwctl search pod-security

# Search for privileged-related policies
kwctl search privileged

# Browse all available policies (paginated)
kwctl search --display-all
```

## Discovering Policies

### Searching for Common Security Policies

```bash
# Find pod security policies
kwctl search "pod security"

# Find image-related policies
kwctl search image

# Find network-related policies
kwctl search network

# Find resource limit policies
kwctl search resource
```

### Getting Policy Details

```bash
# Get metadata about a specific policy
kwctl inspect \
  registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0

# View the full policy documentation
kwctl inspect \
  registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0 \
  --detailed
```

## Evaluating Policies Before Deploying

Before deploying a policy, test it against your existing resources:

```bash
# Download the policy locally
kwctl pull \
  registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0

# List downloaded policies
kwctl policies list

# Test the policy against a manifest
kubectl get pod my-app -n production -o json | \
  kwctl run \
    registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0 \
    --request-path /dev/stdin

# Test with specific settings
kwctl run \
  registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0 \
  --request-path my-pod.json \
  --settings-json '{}'
```

## Popular Hub Policies

### Pod Privileged Policy

Prevents pods from running in privileged mode:

```yaml
# deploy-pod-privileged.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: no-privileged-pods
spec:
  module: registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
```

### Host Namespaces Policy

Prevents pods from using host networking, PID, and IPC namespaces:

```yaml
# deploy-host-namespaces.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: no-host-namespaces
spec:
  module: registry://ghcr.io/kubewarden/policies/host-namespaces-psp:v0.1.1
  settings:
    hostPID: false
    hostIPC: false
    hostNetwork: false
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
```

### Allowed Image Repositories Policy

Restricts images to approved registries:

```yaml
# deploy-allowed-registries.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: allowed-registries
spec:
  module: registry://ghcr.io/kubewarden/policies/allowed-image-repositories:v0.1.0
  settings:
    allowedRegistries:
      - registry.internal.example.com
      - gcr.io/my-org
      - docker.io/library
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
```

### Safe Annotations Policy

Prevents modification of system annotations:

```yaml
# deploy-safe-annotations.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: safe-annotations
spec:
  module: registry://ghcr.io/kubewarden/policies/safe-annotations:v0.2.0
  settings:
    deniedAnnotations:
      - "kubernetes.io/cluster-service"
      - "scheduler.alpha.kubernetes.io/critical-pod"
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
```

## Automating Policy Discovery and Deployment

```bash
#!/bin/bash
# deploy-security-baseline.sh
# Deploys a baseline set of security policies from the hub

NAMESPACE="kubewarden"

echo "Deploying Kubewarden security baseline policies..."

# Apply all policies at once
kubectl apply -f - <<EOF
---
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: baseline-no-privileged
spec:
  module: registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: monitor  # Start in monitor mode
---
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: baseline-no-host-namespaces
spec:
  module: registry://ghcr.io/kubewarden/policies/host-namespaces-psp:v0.1.1
  settings:
    hostPID: false
    hostIPC: false
    hostNetwork: false
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: monitor
EOF

echo "Policies deployed in monitor mode. Review violations before switching to protect mode."
```

## Checking Policy Versions

```bash
# List all available versions of a policy
kwctl pull \
  --list-tags \
  registry://ghcr.io/kubewarden/policies/pod-privileged

# Pull a specific version
kwctl pull \
  registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0

# Check for policy updates
kwctl policies list
```

## Conclusion

The Kubewarden Policy Hub dramatically accelerates your security implementation by providing production-ready, community-tested policies that you can deploy immediately. By starting policies in monitor mode, you can see what would be blocked before enabling enforcement, giving you confidence in deploying new policies without disrupting existing workloads. The combination of the hub's policy library and `kwctl`'s testing capabilities provides a complete workflow for discovering, evaluating, and safely deploying admission policies.
