# How to Configure Google Cloud Provider in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, GCP, Google Cloud, Cloud Provider

Description: Configure the Google Cloud cloud provider in Rancher-managed clusters to enable GCP Load Balancers, Persistent Disks, and Filestore integration.

## Introduction

Integrating the Google Cloud cloud provider with Rancher-managed clusters unlocks GCP-native features: automatic Google Cloud Load Balancer provisioning, dynamic Persistent Disk volumes, and Filestore-backed PersistentVolumes. This guide covers the full setup for RKE2 clusters running on GCE instances.

## Prerequisites

- RKE2 cluster running on Google Compute Engine (GCE) VMs, managed by Rancher
- A GCP Service Account with required IAM roles
- `gcloud` CLI configured locally

## Step 1: Create a GCP Service Account

```bash
# Set variables
PROJECT_ID="my-gcp-project"
SA_NAME="rancher-cloud-provider"

# Create the service account
gcloud iam service-accounts create ${SA_NAME} \
  --display-name="Rancher Kubernetes Cloud Provider" \
  --project="${PROJECT_ID}"

SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

# Assign required roles
for role in \
  roles/compute.instanceAdmin.v1 \
  roles/compute.networkAdmin \
  roles/compute.securityAdmin \
  roles/iam.serviceAccountUser; do
  gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
    --member="serviceAccount:${SA_EMAIL}" \
    --role="${role}"
done

# Download the key file
gcloud iam service-accounts keys create \
  /tmp/gcp-cloud-provider-key.json \
  --iam-account="${SA_EMAIL}"
```

## Step 2: Create the Cloud Provider Secret

```bash
# Store the service account key as a Kubernetes Secret
kubectl create secret generic gcp-cloud-provider \
  --from-file=cloud-sa.json=/tmp/gcp-cloud-provider-key.json \
  -n kube-system
```

## Step 3: Configure RKE2 with GCP Cloud Provider

```yaml
# /etc/rancher/rke2/config.yaml (server nodes)
cloud-provider-name: gce
# The GCE cloud provider reads project metadata automatically from the GCE metadata server
# No explicit config file is required when using GCE metadata service
```

For using a service account key file instead of instance metadata:

```yaml
# /etc/rancher/rke2/config.yaml
cloud-provider-name: gce
cloud-provider-config: /etc/rancher/rke2/gce-cloud-config.yaml
```

```yaml
# /etc/rancher/rke2/gce-cloud-config.yaml
[global]
project-id = my-gcp-project
network-project-id = my-gcp-project
network-name = my-vpc
subnetwork-name = my-subnet
node-tags = rancher-cluster-node
node-instance-prefix = cluster-node-
```

## Step 4: Install the GCP Cloud Controller Manager

```bash
# Deploy the GCP CCM DaemonSet
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-gcp/master/deploy/cloud-controller-manager.yaml

# Or via Helm
helm repo add cloud-provider-gcp \
  https://charts.helm.sh/stable
helm install gcp-cloud-controller-manager cloud-provider-gcp/gcp-compute-persistent-disk-csi-driver \
  --namespace kube-system
```

## Step 5: Install the GCE Persistent Disk CSI Driver

```bash
# Install via Helm
helm repo add gcp-compute-persistent-disk-csi-driver \
  https://kubernetes-sigs.github.io/gcp-compute-persistent-disk-csi-driver

helm install gcp-pd-csi-driver \
  gcp-compute-persistent-disk-csi-driver/gcp-compute-persistent-disk-csi-driver \
  --namespace kube-system \
  --set cloudSA.secretName=gcp-cloud-provider \
  --set cloudSA.secretKey=cloud-sa.json
```

## Step 6: Create GCP StorageClasses

```yaml
# gcp-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd          # pd-standard or pd-ssd or pd-balanced
  replication-type: none
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-standard
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-standard
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f gcp-storageclass.yaml
kubectl get storageclass
```

## Step 7: Configure GCP Filestore (NFS) CSI (Optional)

```bash
# Install GCP Filestore CSI Driver for ReadWriteMany volumes
helm repo add gcp-filestore-csi-driver \
  https://kubernetes-sigs.github.io/gcp-filestore-csi-driver

helm install gcp-filestore-csi-driver \
  gcp-filestore-csi-driver/gcp-filestore-csi-driver \
  --namespace kube-system
```

## Step 8: Verify the Integration

```bash
# Test Google Cloud Load Balancer provisioning
kubectl expose deployment nginx \
  --type=LoadBalancer \
  --port=80 \
  --name=gcp-lb-test

kubectl get service gcp-lb-test -w
# EXTERNAL-IP should show a GCP external IP

# Test PVC provisioning
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gcp-pd-test
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gcp-ssd
  resources:
    requests:
      storage: 20Gi
EOF

kubectl get pvc gcp-pd-test -w
```

## Common Issues

| Issue | Cause | Resolution |
|---|---|---|
| `LB IP pending` | Service account missing `compute.networkAdmin` | Add missing IAM role |
| `PVC fails to bind` | CSI driver pod not running | Check `kubectl get pods -n kube-system -l app=gcp-pd-csi-driver` |
| `cannot get instance metadata` | Not running on GCE | Ensure VMs are GCE instances with metadata server accessible |

## Conclusion

The Google Cloud cloud provider integration in Rancher gives you full GCP infrastructure automation from within Kubernetes. With the GCP CCM, PD CSI Driver, and appropriate IAM permissions, your cluster can dynamically provision Google Cloud Load Balancers and Persistent Disks on demand. Use Workload Identity or instance service accounts in production to avoid managing service account key files.
