# How to Create Kubernetes Volume Snapshot Classes with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, Volume Snapshots, OpenTofu, Storage, Backup, Infrastructure

Description: Learn how to create Kubernetes Volume Snapshot Classes with OpenTofu to enable on-demand and scheduled volume snapshots for persistent storage backup and recovery.

## Overview

Kubernetes Volume Snapshot Classes define the provisioner and deletion policy for volume snapshots, similar to how StorageClasses work for PVCs. OpenTofu manages VolumeSnapshotClasses and the snapshot lifecycle.

## Step 1: Install Volume Snapshot CRDs

```hcl
# main.tf - Install VolumeSnapshot CRDs via Helm
resource "helm_release" "snapshot_controller" {
  name             = "snapshot-controller"
  repository       = "https://piraeus.io/helm-charts/"
  chart            = "snapshot-controller"
  namespace        = "kube-system"
  create_namespace = false
  version          = "2.1.0"
}
```

## Step 2: Create VolumeSnapshotClass for GKE (CSI Driver)

```hcl
# VolumeSnapshotClass for GKE persistent disks
resource "kubernetes_manifest" "gke_snapshot_class" {
  manifest = {
    apiVersion = "snapshot.storage.k8s.io/v1"
    kind       = "VolumeSnapshotClass"
    metadata = {
      name = "gke-pd-snapshot-class"
      # Set as default snapshot class
      annotations = {
        "snapshot.storage.kubernetes.io/is-default-class" = "true"
      }
    }
    driver          = "pd.csi.storage.gke.io"
    deletionPolicy  = "Delete"  # "Delete" or "Retain" snapshots

    parameters = {
      "storage-locations" = "us-central1"
    }
  }

  depends_on = [helm_release.snapshot_controller]
}
```

## Step 3: Create VolumeSnapshotClass for AWS EBS

```hcl
# VolumeSnapshotClass for AWS EBS CSI driver
resource "kubernetes_manifest" "ebs_snapshot_class" {
  manifest = {
    apiVersion = "snapshot.storage.k8s.io/v1"
    kind       = "VolumeSnapshotClass"
    metadata = {
      name = "ebs-csi-snapshot-class"
    }
    driver         = "ebs.csi.aws.com"
    deletionPolicy = "Retain"  # Retain snapshots even when VolumeSnapshot object is deleted

    parameters = {
      "tagSpecification_1" = "key=managed-by,value=opentofu"
    }
  }
}
```

## Step 4: Create a Volume Snapshot

```hcl
# Create an on-demand snapshot of a PVC
resource "kubernetes_manifest" "database_snapshot" {
  manifest = {
    apiVersion = "snapshot.storage.k8s.io/v1"
    kind       = "VolumeSnapshot"
    metadata = {
      name      = "postgres-data-snapshot-initial"
      namespace = "production"
    }
    spec = {
      volumeSnapshotClassName = kubernetes_manifest.gke_snapshot_class.manifest.metadata.name
      source = {
        persistentVolumeClaimName = "postgres-data-postgres-0"  # PVC name
      }
    }
  }
}
```

## Step 5: Restore PVC from Snapshot

```hcl
# Create a new PVC from a volume snapshot
resource "kubernetes_persistent_volume_claim_v1" "restored_pvc" {
  metadata {
    name      = "postgres-data-restored"
    namespace = "production"
  }

  spec {
    access_modes = ["ReadWriteOnce"]

    resources {
      requests = {
        storage = "50Gi"
      }
    }

    storage_class_name = "premium-rwo"

    # Restore from snapshot
    data_source {
      name      = kubernetes_manifest.database_snapshot.manifest.metadata.name
      kind      = "VolumeSnapshot"
      api_group = "snapshot.storage.k8s.io"
    }
  }
}
```

## Summary

Kubernetes Volume Snapshot Classes with OpenTofu enable cloud-native backup for persistent storage. Use `deletionPolicy: Retain` for production snapshots that need independent lifecycle from the Kubernetes object. Combine with scheduled Jobs that create VolumeSnapshot resources for automated backups without external backup tools.
