# How to Manage Rook-Ceph Resources with Terraform Kubernetes Provider

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Terraform, Kubernetes, CRD, Infrastructure as Code

Description: Use the Terraform Kubernetes provider to manage Rook-Ceph CRDs and custom resources like CephCluster, CephBlockPool, and CephObjectStore as code.

---

The Terraform Kubernetes provider lets you manage Rook-Ceph custom resources (CRDs) directly as Terraform resources, giving you state tracking, drift detection, and dependency management for your entire Ceph configuration.

## Provider Setup

```hcl
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
  }
}

provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "rook-ceph-cluster"
}
```

## Managing the CephCluster Resource

```hcl
resource "kubernetes_manifest" "ceph_cluster" {
  manifest = {
    apiVersion = "ceph.rook.io/v1"
    kind       = "CephCluster"
    metadata = {
      name      = "rook-ceph"
      namespace = "rook-ceph"
    }
    spec = {
      cephVersion = {
        image            = "quay.io/ceph/ceph:v18.2.0"
        allowUnsupported = false
      }
      dataDirHostPath = "/var/lib/rook"
      mon = {
        count                = 3
        allowMultiplePerNode = false
      }
      mgr = {
        count = 2
        modules = [
          { name = "pg_autoscaler", enabled = true },
          { name = "dashboard", enabled = true }
        ]
      }
      dashboard = {
        enabled = true
        ssl     = true
      }
      storage = {
        useAllNodes   = false
        useAllDevices = false
        nodes = [
          {
            name = "storage-node-1"
            devices = [{ name = "/dev/sdb" }]
          },
          {
            name = "storage-node-2"
            devices = [{ name = "/dev/sdb" }]
          },
          {
            name = "storage-node-3"
            devices = [{ name = "/dev/sdb" }]
          }
        ]
      }
    }
  }
}
```

## Managing CephBlockPool

```hcl
resource "kubernetes_manifest" "ceph_block_pool" {
  manifest = {
    apiVersion = "ceph.rook.io/v1"
    kind       = "CephBlockPool"
    metadata = {
      name      = "replicapool"
      namespace = "rook-ceph"
    }
    spec = {
      failureDomain = "host"
      replicated = {
        size                   = 3
        requireSafeReplicaSize = true
      }
    }
  }

  depends_on = [kubernetes_manifest.ceph_cluster]
}

resource "kubernetes_manifest" "ceph_storageclass" {
  manifest = {
    apiVersion = "storage.k8s.io/v1"
    kind       = "StorageClass"
    metadata = {
      name = "rook-ceph-block"
    }
    provisioner   = "rook-ceph.rbd.csi.ceph.com"
    reclaimPolicy = "Retain"
    parameters = {
      clusterID              = "rook-ceph"
      pool                   = "replicapool"
      imageFormat            = "2"
      imageFeatures          = "layering"
      "csi.storage.k8s.io/provisioner-secret-name"      = "rook-csi-rbd-provisioner"
      "csi.storage.k8s.io/provisioner-secret-namespace" = "rook-ceph"
      "csi.storage.k8s.io/node-stage-secret-name"       = "rook-csi-rbd-node"
      "csi.storage.k8s.io/node-stage-secret-namespace"  = "rook-ceph"
    }
  }

  depends_on = [kubernetes_manifest.ceph_block_pool]
}
```

## Managing CephObjectStore

```hcl
resource "kubernetes_manifest" "ceph_object_store" {
  manifest = {
    apiVersion = "ceph.rook.io/v1"
    kind       = "CephObjectStore"
    metadata = {
      name      = "my-store"
      namespace = "rook-ceph"
    }
    spec = {
      metadataPool = {
        replicated = { size = 3 }
      }
      dataPool = {
        replicated = { size = 3 }
      }
      preservePoolsOnDelete = false
      gateway = {
        port      = 80
        instances = 2
      }
    }
  }

  depends_on = [kubernetes_manifest.ceph_cluster]
}
```

## Using terraform_data for Dependencies

```hcl
# Wait for Rook operator to be ready before applying CRDs
resource "terraform_data" "wait_for_operator" {
  provisioner "local-exec" {
    command = <<-EOT
      kubectl -n rook-ceph wait deploy/rook-ceph-operator \
        --for=condition=Available --timeout=300s
    EOT
  }
}

resource "kubernetes_manifest" "ceph_cluster" {
  # ... cluster spec ...
  depends_on = [terraform_data.wait_for_operator]
}
```

## Drift Detection

```bash
# Check for drift between Terraform state and actual cluster config
terraform plan

# If drift detected, reconcile
terraform apply

# Import existing Rook resources into Terraform state
terraform import kubernetes_manifest.ceph_cluster \
  "apiVersion=ceph.rook.io/v1,kind=CephCluster,namespace=rook-ceph,name=rook-ceph"
```

## Summary

Managing Rook-Ceph resources with the Terraform Kubernetes provider brings infrastructure-as-code benefits to your entire Ceph configuration, including CRDs like CephCluster, CephBlockPool, and CephObjectStore. Terraform's state management detects configuration drift and the dependency graph ensures resources are created in the correct order.
