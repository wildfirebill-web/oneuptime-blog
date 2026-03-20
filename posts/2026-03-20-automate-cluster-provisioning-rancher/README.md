# How to Automate Cluster Provisioning in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Cluster Provisioning, Automation, DevOps, Infrastructure

Description: Learn how to automate Kubernetes cluster provisioning in Rancher using the Rancher API, Terraform provider, and cluster templates.

---

Rancher supports automating cluster provisioning through its API and the official Rancher Terraform/OpenTofu provider. This enables repeatable, version-controlled cluster creation across cloud providers and bare metal.

---

## Install the Rancher Provider

```hcl
# versions.tf
terraform {
  required_providers {
    rancher2 = {
      source  = "rancher/rancher2"
      version = "~> 4.1"
    }
  }
}

provider "rancher2" {
  api_url    = "https://rancher.example.com"
  access_key = var.rancher_access_key
  secret_key = var.rancher_secret_key
  insecure   = false
}
```

---

## Provision an RKE2 Cluster on AWS

```hcl
resource "rancher2_machine_config_v2" "worker" {
  generate_name = "worker-"
  amazonec2_config {
    ami                = "ami-0c55b159cbfafe1f0"
    region             = "us-east-1"
    instance_type      = "t3.medium"
    security_group     = [aws_security_group.rancher_nodes.name]
    subnet_id          = aws_subnet.private.id
    vpc_id             = aws_vpc.main.id
    zone               = "a"
  }
}

resource "rancher2_cluster_v2" "prod" {
  name               = "prod-cluster"
  kubernetes_version = "v1.28.8+rke2r1"

  rke_config {
    machine_pools {
      name                         = "control-plane"
      cloud_credential_secret_name = rancher2_cloud_credential.aws.name
      control_plane_role           = true
      etcd_role                    = true
      worker_role                  = false
      quantity                     = 3
      machine_config {
        kind = rancher2_machine_config_v2.worker.kind
        name = rancher2_machine_config_v2.worker.name
      }
    }

    machine_pools {
      name                         = "workers"
      cloud_credential_secret_name = rancher2_cloud_credential.aws.name
      control_plane_role           = false
      etcd_role                    = false
      worker_role                  = true
      quantity                     = 3
      machine_config {
        kind = rancher2_machine_config_v2.worker.kind
        name = rancher2_machine_config_v2.worker.name
      }
    }
  }
}
```

---

## Create Cloud Credentials

```hcl
resource "rancher2_cloud_credential" "aws" {
  name = "aws-credentials"
  amazonec2_credential_config {
    access_key = var.aws_access_key
    secret_key = var.aws_secret_key
  }
}
```

---

## Use the Rancher API Directly

```bash
# Create a cluster via the Rancher API
curl -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  https://rancher.example.com/v3/clusters \
  -d '{
    "name": "dev-cluster",
    "rkeConfig": {
      "kubernetesVersion": "v1.28.8+rke2r1"
    }
  }'
```

---

## Get Kubeconfig for the Cluster

```bash
# Using Rancher CLI
rancher login https://rancher.example.com --token ${RANCHER_TOKEN}
rancher clusters kubeconfig prod-cluster > ~/.kube/prod-config
```

---

## Summary

Use the `rancher/rancher2` OpenTofu provider to declare clusters as code with `rancher2_cluster_v2`. Define separate machine pools for control plane and worker roles, attach cloud credentials via `rancher2_cloud_credential`, and apply with `tofu apply`. This approach makes cluster provisioning repeatable, reviewable through pull requests, and consistent across environments.
