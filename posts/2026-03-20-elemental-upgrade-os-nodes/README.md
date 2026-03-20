# How to Upgrade Elemental OS on Nodes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Kubernetes, Upgrades, Edge, OS Management

Description: A complete guide to upgrading the Elemental OS on registered nodes using ManagedOSImage resources for rolling, controlled updates across your fleet.

## Introduction

One of Elemental's most powerful features is its support for zero-downtime, declarative OS upgrades. Rather than manually SSHing into nodes, you create a ManagedOSImage resource that targets specific machines and the Elemental Operator orchestrates the upgrade process, including reboots, in a controlled manner.

## Prerequisites

- Elemental Operator installed
- Machines registered in MachineInventory
- New OS container image pushed to a registry
- Target machines running Elemental OS

## Step 1: Build and Push the New OS Image

```bash
# Build the new OS version
docker build \
  -t my-registry.example.com/elemental-os:v1.1.0 \
  -f Dockerfile.elemental \
  .

# Push to registry
docker push my-registry.example.com/elemental-os:v1.1.0
```

## Step 2: Create a ManagedOSImage Resource

The ManagedOSImage resource defines which machines to upgrade and to which OS version:

```yaml
# managed-os-upgrade.yaml
apiVersion: elemental.cattle.io/v1beta1
kind: ManagedOSImage
metadata:
  name: upgrade-to-v1.1.0
  namespace: fleet-default
spec:
  # The new OS container image
  osImage: "my-registry.example.com/elemental-os:v1.1.0"

  # Target all machines, or use a selector for targeted upgrades
  clusterSelector:
    matchLabels:
      # Target clusters with this label
      environment: production

  # Node selector within the cluster
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/worker: "true"

  # Concurrency settings
  concurrency: 2

  # Cordon nodes before upgrade
  cordon: true

  # Drain nodes before upgrade
  drain:
    enabled: true
    force: false
    timeout: 300
    gracePeriod: 30
    ignoreDaemonSets: true
    deleteLocalData: false
```

```bash
kubectl apply -f managed-os-upgrade.yaml
```

## Step 3: Monitor Upgrade Progress

```bash
# Watch the ManagedOSImage status
kubectl get managedosimage -n fleet-default upgrade-to-v1.1.0 --watch

# Get detailed status
kubectl describe managedosimage -n fleet-default upgrade-to-v1.1.0

# Check upgrade jobs
kubectl get jobs -n cattle-system -l upgrade.cattle.io/managed-os-image=upgrade-to-v1.1.0

# Watch pods during upgrade
kubectl get pods -n cattle-system -l upgrade.cattle.io/managed-os-image=upgrade-to-v1.1.0 --watch
```

## Step 4: Verify the Upgrade

```bash
# SSH into a node and check OS version
ssh root@node-ip cat /etc/os-release

# Check via kubectl
kubectl get node <node-name> -o jsonpath='{.metadata.annotations}'

# Verify all nodes are at the new version
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,OS-IMAGE:.status.nodeInfo.osImage'
```

## Rolling Upgrade Strategy

For production environments, use a rolling upgrade approach:

```yaml
# rolling-upgrade.yaml
apiVersion: elemental.cattle.io/v1beta1
kind: ManagedOSImage
metadata:
  name: rolling-upgrade-v1.1.0
  namespace: fleet-default
spec:
  osImage: "my-registry.example.com/elemental-os:v1.1.0"

  # Only upgrade worker nodes initially
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/worker: "true"

  # Upgrade one node at a time
  concurrency: 1

  # Wait 60 seconds between node upgrades
  drain:
    enabled: true
    timeout: 600
```

## Upgrading via ManagedOSVersionChannel

```yaml
# Use version channels for managed rollouts
apiVersion: elemental.cattle.io/v1beta1
kind: ManagedOSImage
metadata:
  name: channel-upgrade
  namespace: fleet-default
spec:
  # Reference a version channel instead of a direct image
  managedOSVersionName: elemental-v1.1.0
```

## Rollback on Failure

Elemental OS uses A/B partition schemes, enabling rollback if an upgrade fails:

```bash
# If a node fails to boot after upgrade, it will auto-rollback
# To manually trigger rollback on a node:
ssh root@node-ip grub2-once recovery

# Or trigger rollback via the system
ssh root@node-ip elemental reset
```

## Conclusion

Elemental's declarative OS upgrade system transforms the traditionally painful process of updating bare metal nodes into a Kubernetes-native operation. ManagedOSImage resources allow you to target specific machines, control upgrade concurrency, and leverage drain/cordon for zero-downtime upgrades. Combined with A/B partition rollback, Elemental OS upgrades are safe, repeatable, and auditable.
