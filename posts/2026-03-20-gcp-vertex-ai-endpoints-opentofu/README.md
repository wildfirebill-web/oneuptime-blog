# How to Create GCP Vertex AI Endpoints with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Vertex AI, Machine Learning, MLOps, Infrastructure as Code

Description: Learn how to create GCP Vertex AI endpoints, deploy models, and configure traffic splits for A/B testing using OpenTofu.

## Introduction

Vertex AI Endpoints serve trained ML models for online prediction. You can deploy multiple model versions to a single endpoint and split traffic between them for gradual rollouts or A/B testing. OpenTofu manages endpoints and model deployments as code.

## Enabling Required APIs

```hcl
resource "google_project_service" "vertex_ai" {
  project = var.project_id
  service = "aiplatform.googleapis.com"
}

resource "google_project_service" "container_registry" {
  project = var.project_id
  service = "containerregistry.googleapis.com"
}
```

## Service Account for Vertex AI

```hcl
resource "google_service_account" "vertex_ai" {
  account_id   = "${var.app_name}-vertex-ai-sa"
  display_name = "Vertex AI Service Account"
  project      = var.project_id
}

resource "google_project_iam_member" "vertex_ai_user" {
  project = var.project_id
  role    = "roles/aiplatform.user"
  member  = "serviceAccount:${google_service_account.vertex_ai.email}"
}
```

## Creating a Vertex AI Endpoint

```hcl
resource "google_vertex_ai_endpoint" "prediction" {
  name         = "${var.app_name}-prediction-endpoint"
  display_name = "${var.app_name} Prediction Endpoint"
  project      = var.project_id
  location     = var.region
  description  = "Online prediction endpoint for ${var.app_name}"

  labels = {
    environment = var.environment
    managed_by  = "opentofu"
  }
}
```

## Uploading a Model

```hcl
resource "google_vertex_ai_model" "classifier" {
  display_name = "${var.app_name}-classifier-v${var.model_version}"
  project      = var.project_id
  location     = var.region

  artifact_uri = "gs://${var.model_bucket}/models/classifier/v${var.model_version}/"

  container_spec {
    image_uri = "us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-0:latest"

    # Optionally specify a custom serving command
    # command = ["python", "serve.py"]

    env {
      name  = "MODEL_VERSION"
      value = var.model_version
    }
  }

  labels = {
    environment = var.environment
  }
}
```

## Deploying Model to Endpoint

```hcl
resource "google_vertex_ai_endpoint_deployment" "primary" {
  endpoint = google_vertex_ai_endpoint.prediction.name
  project  = var.project_id
  location = var.region

  deployed_models {
    model                  = google_vertex_ai_model.classifier.id
    display_name           = "classifier-primary"
    traffic_split          = 90  # 90% of traffic to primary

    dedicated_resources {
      min_replica_count = 1
      max_replica_count = 5

      machine_spec {
        machine_type = "n1-standard-4"
      }

      autoscaling_metric_specs {
        metric_name = "aiplatform.googleapis.com/prediction/online/cpu/utilization"
        target      = 60  # scale when CPU > 60%
      }
    }
  }

  deployed_models {
    model        = google_vertex_ai_model.classifier_canary.id
    display_name = "classifier-canary"
    traffic_split = 10  # 10% of traffic to canary

    dedicated_resources {
      min_replica_count = 1
      max_replica_count = 2

      machine_spec {
        machine_type = "n1-standard-4"
      }
    }
  }
}
```

## Outputs

```hcl
output "endpoint_id" {
  value = google_vertex_ai_endpoint.prediction.id
}

output "endpoint_name" {
  value = google_vertex_ai_endpoint.prediction.name
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

Vertex AI endpoints provide managed online prediction with traffic splitting for safe model rollouts. OpenTofu manages endpoint creation, model registration, and deployment configurations — enabling reproducible, version-controlled ML model serving.
