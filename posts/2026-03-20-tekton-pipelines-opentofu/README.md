# How to Deploy Tekton Pipelines with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Tekton, CI/CD, Kubernetes, Pipeline, Infrastructure as Code, Cloud Native

Description: Learn how to deploy Tekton Pipelines on Kubernetes using OpenTofu to enable cloud-native CI/CD with reusable Tasks and Pipelines defined as Kubernetes resources.

---

Tekton is a Kubernetes-native CI/CD framework where pipelines are first-class Kubernetes resources. Every build runs in a pod, making it portable, scalable, and cloud-agnostic. OpenTofu handles the Tekton installation and bootstrap configuration.

## Deploying Tekton Pipelines

```hcl
# main.tf

terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.24"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}

provider "kubernetes" {
  host                   = var.cluster_endpoint
  cluster_ca_certificate = base64decode(var.cluster_ca_cert)
  token                  = var.cluster_token
}

# Install Tekton Pipelines from the official release manifest
resource "null_resource" "tekton_install" {
  provisioner "local-exec" {
    command = "kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml"
  }
}

# Install Tekton Dashboard for visualization
resource "null_resource" "tekton_dashboard" {
  depends_on = [null_resource.tekton_install]

  provisioner "local-exec" {
    command = "kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml"
  }
}
```

## Using the Helm Chart (Alternative Approach)

```hcl
# tekton_helm.tf
resource "kubernetes_namespace" "tekton" {
  metadata {
    name = "tekton-pipelines"
  }
}

resource "helm_release" "tekton_pipeline" {
  name       = "tekton-pipeline"
  repository = "https://charts.tekton.dev"
  chart      = "tekton-pipeline"
  namespace  = kubernetes_namespace.tekton.metadata[0].name

  wait    = true
  timeout = 300

  values = [
    yamlencode({
      # Tekton pipeline configuration
      pipeline = {
        metrics = {
          enabled = true
        }
      }
    })
  ]
}
```

## Creating a Tekton Task

```hcl
# tasks.tf
# A reusable task for building and pushing Docker images
resource "kubernetes_manifest" "build_push_task" {
  depends_on = [null_resource.tekton_install]

  manifest = {
    apiVersion = "tekton.dev/v1"
    kind       = "Task"
    metadata = {
      name      = "build-and-push"
      namespace = "default"
    }
    spec = {
      params = [
        {
          name        = "IMAGE"
          description = "The image to build and push"
          type        = "string"
        },
        {
          name        = "DOCKERFILE"
          description = "Path to the Dockerfile"
          type        = "string"
          default     = "./Dockerfile"
        }
      ]
      workspaces = [
        {
          name        = "source"
          description = "The workspace containing the cloned source code"
        }
      ]
      steps = [
        {
          name  = "build-and-push"
          image = "gcr.io/kaniko-project/executor:latest"
          args  = [
            "--dockerfile=$(params.DOCKERFILE)",
            "--context=dir://$(workspaces.source.path)",
            "--destination=$(params.IMAGE)"
          ]
          volumeMounts = [
            {
              name      = "docker-credentials"
              mountPath = "/kaniko/.docker"
            }
          ]
        }
      ]
      volumes = [
        {
          name = "docker-credentials"
          secret = {
            secretName = "docker-credentials"
          }
        }
      ]
    }
  }
}
```

## Creating a Pipeline

```hcl
# pipeline.tf
resource "kubernetes_manifest" "ci_pipeline" {
  depends_on = [kubernetes_manifest.build_push_task]

  manifest = {
    apiVersion = "tekton.dev/v1"
    kind       = "Pipeline"
    metadata = {
      name      = "ci-pipeline"
      namespace = "default"
    }
    spec = {
      params = [
        {
          name = "repo-url"
          type = "string"
        },
        {
          name = "image"
          type = "string"
        }
      ]
      workspaces = [
        {
          name = "shared-workspace"
        }
      ]
      tasks = [
        {
          name = "fetch-source"
          taskRef = {
            name = "git-clone"
          }
          workspaces = [
            {
              name      = "output"
              workspace = "shared-workspace"
            }
          ]
          params = [
            { name = "url", value = "$(params.repo-url)" }
          ]
        },
        {
          name    = "build-and-push"
          runAfter = ["fetch-source"]
          taskRef = {
            name = "build-and-push"
          }
          workspaces = [
            {
              name      = "source"
              workspace = "shared-workspace"
            }
          ]
          params = [
            { name = "IMAGE", value = "$(params.image)" }
          ]
        }
      ]
    }
  }
}
```

## Best Practices

- Use the Tekton Hub (hub.tekton.dev) to find community-maintained tasks rather than writing everything from scratch.
- Use workspaces for sharing data between tasks - avoid embedding source paths in task parameters.
- Configure resource limits on all task steps to prevent runaway builds from consuming cluster resources.
- Use Tekton Chains for software supply chain security - it automatically signs your pipeline artifacts.
- Store sensitive pipeline secrets (registry credentials, signing keys) as Kubernetes secrets referenced by tasks.
