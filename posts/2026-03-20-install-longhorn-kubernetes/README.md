# How to Install Longhorn on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, Installation

Description: A comprehensive guide to installing Longhorn, the cloud-native distributed block storage system, on a Kubernetes cluster.

## Introduction

Longhorn is a lightweight, reliable, and powerful distributed block storage system for Kubernetes. Developed by Rancher Labs and now a CNCF project, Longhorn provides persistent storage for stateful applications in Kubernetes environments. This guide walks you through the complete installation process.

## Prerequisites

Before installing Longhorn, ensure your cluster meets the following requirements:

- Kubernetes version 1.21 or later
- Each node must have the following utilities installed: `open-iscsi`, `util-linux`, `bash`, `curl`, `findmnt`, `grep`, `awk`, `blkid`, `lsblk`
- Minimum recommended hardware: 2 vCPU, 4 GiB RAM per node
- Container runtime compatible with Longhorn (Docker, containerd, CRI-O)

### Verify Prerequisites

Longhorn provides a script to check if your environment meets all prerequisites:

```bash
# Download and run the environment check script

curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.7.0/scripts/environment_check.sh | bash
```

You should see output indicating whether each node passes or fails the checks.

### Install Required Packages

On each Kubernetes node (for Debian/Ubuntu):

```bash
# Install open-iscsi and nfs-common
apt-get install -y open-iscsi nfs-common

# Enable and start the iscsid service
systemctl enable iscsid
systemctl start iscsid
```

For RHEL/CentOS:

```bash
# Install required packages
yum install -y iscsi-initiator-utils nfs-utils

# Enable and start the iscsid service
systemctl enable iscsid
systemctl start iscsid
```

## Installation Methods

Longhorn can be installed using several methods. This guide covers the primary `kubectl apply` approach. For Helm-based installation, refer to the dedicated Helm guide.

### Method 1: Install Using kubectl

The simplest way to install Longhorn is to apply the official manifest directly:

```bash
# Install Longhorn using the official manifest
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.0/deploy/longhorn.yaml
```

This single command installs all Longhorn components in the `longhorn-system` namespace.

### Verify the Installation

Monitor the rollout of Longhorn components:

```bash
# Watch the pods come up in the longhorn-system namespace
kubectl get pods -n longhorn-system -w
```

Wait until all pods show a status of `Running`. This may take a few minutes depending on your network speed and cluster resources.

```bash
# Check that all Longhorn components are running
kubectl get pods -n longhorn-system
```

Expected output includes pods for:
- `longhorn-manager` (one per node)
- `longhorn-driver-deployer`
- `instance-manager`
- `engine-image`

### Access the Longhorn UI

By default, the Longhorn frontend is exposed as a `ClusterIP` service. To access it, use port forwarding:

```bash
# Port-forward the Longhorn frontend service to localhost
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
```

Now open your browser and navigate to `http://localhost:8080` to access the Longhorn UI.

## Setting Longhorn as Default Storage Class

After installation, you may want to set Longhorn as your default storage class:

```bash
# Patch the longhorn StorageClass to be the default
kubectl patch storageclass longhorn \
  -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
```

If another storage class is currently the default, remove that annotation first:

```bash
# Remove default annotation from existing default storage class
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'
```

## Create a Test PersistentVolumeClaim

Verify Longhorn works by creating a test PVC:

```yaml
# test-pvc.yaml - A simple PVC using the Longhorn storage class
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

```bash
# Apply the test PVC
kubectl apply -f test-pvc.yaml

# Check the PVC status (should become Bound)
kubectl get pvc longhorn-test-pvc
```

## Monitoring Installation Status

You can use the Longhorn UI or CLI to confirm the installation is healthy:

```bash
# Check all nodes are detected by Longhorn
kubectl get nodes.longhorn.io -n longhorn-system

# Check volume status
kubectl get volumes.longhorn.io -n longhorn-system
```

## Conclusion

You have successfully installed Longhorn on your Kubernetes cluster. Longhorn is now ready to provide distributed persistent storage for your workloads. You can manage volumes, configure backups, and monitor storage health through the Longhorn UI at any time. For production deployments, consider configuring backup targets, replica counts, and resource quotas to ensure reliability and performance.
