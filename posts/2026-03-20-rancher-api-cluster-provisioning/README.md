# How to Automate Cluster Provisioning with Rancher API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, API, Automation, Cluster-provisioning, Terraform

Description: A practical guide to automating Kubernetes cluster provisioning using the Rancher API and Terraform, covering authentication, cluster creation, and post-provisioning configuration.

## Overview

Manually creating clusters through the Rancher UI is fine for occasional setups, but production environments require repeatable, automated provisioning. Rancher provides a comprehensive REST API and supports Terraform through the Rancher2 provider. This guide covers automating cluster provisioning using the Rancher API directly and through Terraform.

## Authentication

### Create an API Key

First, create a Rancher API key:

```text
Rancher UI → User Avatar → API Keys → Add Key
- Description: automation-key
- Expires: (set appropriate expiry)
- Scope: No scope restriction (or restrict to specific clusters)
```

```bash
# Store credentials

export RANCHER_URL="https://rancher.example.com"
export RANCHER_ACCESS_KEY="token-xxxxx"
export RANCHER_SECRET_KEY="xxxxxxxxxxxxxxxxxx"
export RANCHER_TOKEN="${RANCHER_ACCESS_KEY}:${RANCHER_SECRET_KEY}"
```

## Using the Rancher API Directly

### List Existing Clusters

```bash
# List all clusters
curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/clusters" \
  | jq '.data[] | {id: .id, name: .name, state: .state}'
```

### Provision an RKE2 Cluster on vSphere

```bash
# Create an RKE2 cluster via Rancher API
curl -s -k \
  -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  "${RANCHER_URL}/v1/provisioning.cattle.io.clusters" \
  -d '{
    "apiVersion": "provisioning.cattle.io/v1",
    "kind": "Cluster",
    "metadata": {
      "name": "prod-cluster-01",
      "namespace": "fleet-default",
      "labels": {
        "env": "production",
        "region": "us-east"
      }
    },
    "spec": {
      "kubernetesVersion": "v1.28.6+rke2r1",
      "rkeConfig": {
        "nodePools": [
          {
            "name": "control-plane",
            "count": 3,
            "controlPlaneRole": true,
            "etcdRole": true,
            "workerRole": false,
            "nodeConfig": {
              "vmSize": "Standard_D4s_v3",
              "diskSize": 50
            }
          },
          {
            "name": "workers",
            "count": 5,
            "workerRole": true,
            "nodeConfig": {
              "vmSize": "Standard_D8s_v3",
              "diskSize": 100
            }
          }
        ]
      }
    }
  }'
```

### Wait for Cluster to Be Active

```bash
#!/bin/bash
# wait-for-cluster.sh
CLUSTER_ID="$1"

echo "Waiting for cluster ${CLUSTER_ID} to become active..."
while true; do
  STATE=$(curl -s -k \
    -H "Authorization: Bearer ${RANCHER_TOKEN}" \
    "${RANCHER_URL}/v3/clusters/${CLUSTER_ID}" \
    | jq -r '.state')

  echo "Cluster state: ${STATE}"

  if [ "${STATE}" = "active" ]; then
    echo "Cluster is active!"
    break
  elif [ "${STATE}" = "error" ]; then
    echo "Cluster provisioning failed!"
    exit 1
  fi
  sleep 30
done
```

## Using Terraform with the Rancher2 Provider

### Provider Configuration

```hcl
# provider.tf
terraform {
  required_providers {
    rancher2 = {
      source  = "rancher/rancher2"
      version = "~> 4.0"
    }
  }
}

provider "rancher2" {
  api_url    = var.rancher_url
  access_key = var.rancher_access_key
  secret_key = var.rancher_secret_key
}
```

### Cluster Resource

```hcl
# cluster.tf
resource "rancher2_cluster_v2" "prod_cluster" {
  name                  = "prod-cluster-${var.environment}"
  kubernetes_version    = "v1.28.6+rke2r1"
  cloud_credential_secret_name = rancher2_cloud_credential.aws.id

  rke_config {
    machine_pools {
      name                         = "control-plane"
      cloud_credential_secret_name = rancher2_cloud_credential.aws.id
      control_plane_role           = true
      etcd_role                    = true
      worker_role                  = false
      quantity                     = 3

      machine_config {
        kind = "Amazonec2Config"
        name = rancher2_machine_config_v2.control_plane.name
      }
    }

    machine_pools {
      name                         = "workers"
      cloud_credential_secret_name = rancher2_cloud_credential.aws.id
      control_plane_role           = false
      etcd_role                    = false
      worker_role                  = true
      quantity                     = var.worker_count

      machine_config {
        kind = "Amazonec2Config"
        name = rancher2_machine_config_v2.worker.name
      }
    }

    machine_global_config = yamlencode({
      cni                   = "calico"
      secrets-encryption    = true
      profile               = "cis-1.23"
    })
  }

  labels = {
    environment = var.environment
    region      = var.aws_region
    managed-by  = "terraform"
  }
}

# Output the kubeconfig
output "kubeconfig" {
  value     = rancher2_cluster_v2.prod_cluster.kube_config
  sensitive = true
}
```

### Machine Configuration

```hcl
# machine-config.tf
resource "rancher2_machine_config_v2" "worker" {
  generate_name = "worker-${var.environment}"
  resource_version = "1"

  amazonec2_config {
    ami            = var.worker_ami_id
    region         = var.aws_region
    instance_type  = "m5.2xlarge"
    root_size      = "100"
    vpc_id         = var.vpc_id
    subnet_id      = var.private_subnet_id
    security_group = [var.worker_sg_id]
    iam_instance_profile = var.worker_iam_profile
    tags           = "env,${var.environment},managed-by,terraform"
  }
}
```

### Post-Provisioning Configuration

```hcl
# After cluster is active, install monitoring
resource "rancher2_app_v2" "monitoring" {
  cluster_id    = rancher2_cluster_v2.prod_cluster.cluster_v1_id
  name          = "rancher-monitoring"
  namespace     = "cattle-monitoring-system"
  repo_name     = "rancher-charts"
  chart_name    = "rancher-monitoring"
  chart_version = "103.0.0+up45.31.1"

  values = yamlencode({
    prometheus = {
      prometheusSpec = {
        retention     = "30d"
        storageSpec = {
          volumeClaimTemplate = {
            spec = {
              storageClassName = "longhorn"
              resources = {
                requests = { storage = "50Gi" }
              }
            }
          }
        }
      }
    }
  })

  depends_on = [rancher2_cluster_v2.prod_cluster]
}
```

## Automating with GitHub Actions

```yaml
# .github/workflows/provision-cluster.yml
name: Provision Kubernetes Cluster
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [dev, staging, production]
      worker_count:
        description: 'Number of worker nodes'
        required: true
        default: '3'

jobs:
  provision:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init
        working-directory: ./infrastructure/clusters

      - name: Terraform Apply
        env:
          TF_VAR_rancher_url: ${{ secrets.RANCHER_URL }}
          TF_VAR_rancher_access_key: ${{ secrets.RANCHER_ACCESS_KEY }}
          TF_VAR_rancher_secret_key: ${{ secrets.RANCHER_SECRET_KEY }}
          TF_VAR_environment: ${{ inputs.environment }}
          TF_VAR_worker_count: ${{ inputs.worker_count }}
        run: terraform apply -auto-approve
        working-directory: ./infrastructure/clusters
```

## Conclusion

Automating cluster provisioning with the Rancher API and Terraform enables repeatable, auditable infrastructure creation. The Rancher2 Terraform provider handles the full lifecycle of clusters, machine configurations, cloud credentials, and post-provisioning app installations. Combining Terraform with GitHub Actions provides a complete GitOps-style cluster provisioning pipeline where all changes are code-reviewed and version-controlled.
