# How to Deploy Ceph with Terraform on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Terraform, AWS, Infrastructure as Code, Kubernetes

Description: Use Terraform to provision an AWS EKS cluster with EBS volumes and deploy Rook-Ceph for production-grade distributed storage on Amazon Web Services.

---

Deploying Rook-Ceph on AWS with Terraform gives you reproducible, version-controlled infrastructure. This guide provisions an EKS cluster, EBS-backed storage nodes, and installs Rook-Ceph using Terraform.

## Project Structure

```
ceph-aws/
  main.tf
  variables.tf
  outputs.tf
  modules/
    eks/
    rook-ceph/
```

## EKS Cluster with Storage Nodes

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  node_security_group_additional_rules = {
    ingress_ceph_mon = {
      description = "Ceph monitor port"
      protocol    = "tcp"
      from_port   = 6789
      to_port     = 6789
      type        = "ingress"
      self        = true
    }
    ingress_ceph_osd = {
      description = "Ceph OSD ports"
      protocol    = "tcp"
      from_port   = 6800
      to_port     = 7300
      type        = "ingress"
      self        = true
    }
  }

  eks_managed_node_groups = {
    ceph_storage = {
      name           = "ceph-storage"
      instance_types = ["i3.2xlarge"]
      min_size       = 3
      max_size       = 6
      desired_size   = 3

      labels = {
        role = "ceph-storage"
      }

      taints = [{
        key    = "dedicated"
        value  = "ceph"
        effect = "NO_SCHEDULE"
      }]

      block_device_mappings = {
        xvda = {
          device_name = "/dev/xvda"
          ebs = {
            volume_size           = 100
            volume_type           = "gp3"
            delete_on_termination = true
          }
        }
        xvdb = {
          device_name = "/dev/xvdb"
          ebs = {
            volume_size           = 1000
            volume_type           = "gp3"
            iops                  = 3000
            throughput            = 250
            delete_on_termination = false
            encrypted             = true
          }
        }
      }
    }
  }
}
```

## Variables Definition

```hcl
# variables.tf
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
  default     = "rook-ceph-prod"
}

variable "ceph_osd_count" {
  description = "Number of OSD nodes"
  type        = number
  default     = 3
}
```

## Rook-Ceph Helm Installation

```hcl
# rook-ceph helm release
resource "helm_release" "rook_operator" {
  name             = "rook-ceph"
  repository       = "https://charts.rook.io/release"
  chart            = "rook-ceph"
  version          = "v1.13.0"
  namespace        = "rook-ceph"
  create_namespace = true

  set {
    name  = "csi.enableRbdDriver"
    value = "true"
  }

  set {
    name  = "csi.enableCephfsDriver"
    value = "true"
  }

  depends_on = [module.eks]
}

resource "helm_release" "rook_cluster" {
  name       = "rook-ceph-cluster"
  repository = "https://charts.rook.io/release"
  chart      = "rook-ceph-cluster"
  version    = "v1.13.0"
  namespace  = "rook-ceph"

  values = [
    templatefile("${path.module}/ceph-cluster-values.yaml", {
      osd_count = var.ceph_osd_count
    })
  ]

  depends_on = [helm_release.rook_operator]
}
```

## Outputs

```hcl
# outputs.tf
output "eks_cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "kubeconfig_command" {
  value = "aws eks update-kubeconfig --region ${var.aws_region} --name ${var.cluster_name}"
}
```

## Deploying

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan

# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name rook-ceph-prod

# Verify Rook deployment
kubectl -n rook-ceph get pods
```

## Summary

Deploying Rook-Ceph on AWS with Terraform provisions EKS with dedicated i3 storage nodes, EBS volumes with encryption, and installs Rook via Helm - all in a reproducible, version-controlled workflow. Using instance store or dedicated EBS volumes for OSDs ensures the storage performance Ceph requires while maintaining the flexibility of cloud infrastructure.
