# How to Create GCP API Gateway with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, API Gateway, Cloud Run, Cloud Functions, Infrastructure as Code

Description: Learn how to create GCP API Gateway with OpenTofu to expose Cloud Run services and Cloud Functions as managed APIs with authentication, quota limits, and monitoring.

GCP API Gateway provides a fully managed API gateway that handles routing, authentication, and quota management for APIs backed by Cloud Run, Cloud Functions, and App Engine. Managing API Gateway in OpenTofu ensures consistent API configuration alongside your backend service deployments.

## Provider Configuration

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

## API Gateway for Cloud Run

```hcl
# Enable required APIs
resource "google_project_service" "apigateway" {
  service            = "apigateway.googleapis.com"
  disable_on_destroy = false
}

resource "google_project_service" "servicemanagement" {
  service            = "servicemanagement.googleapis.com"
  disable_on_destroy = false
}

resource "google_project_service" "servicecontrol" {
  service            = "servicecontrol.googleapis.com"
  disable_on_destroy = false
}

# Create the API
resource "google_api_gateway_api" "main" {
  provider = google-beta
  api_id   = "${var.service_name}-api"
  project  = var.project_id

  depends_on = [google_project_service.apigateway]
}

# API configuration with OpenAPI spec
resource "google_api_gateway_api_config" "main" {
  provider      = google-beta
  api           = google_api_gateway_api.main.api_id
  api_config_id = "${var.service_name}-config-v1"
  project       = var.project_id

  openapi_documents {
    document {
      path     = "spec.yaml"
      contents = base64encode(templatefile("${path.module}/api-spec.yaml", {
        cloud_run_url = google_cloud_run_v2_service.api.uri
        project_id    = var.project_id
      }))
    }
  }

  gateway_config {
    backend_config {
      google_service_account = google_service_account.api_gateway.email
    }
  }

  lifecycle {
    create_before_destroy = true
  }
}

# Gateway deployment
resource "google_api_gateway_gateway" "main" {
  provider   = google-beta
  api_config = google_api_gateway_api_config.main.id
  gateway_id = "${var.service_name}-gateway"
  region     = var.region
  project    = var.project_id

  display_name = "${var.service_name} API Gateway"
}
```

## OpenAPI Spec with Authentication

The OpenAPI spec file (`api-spec.yaml`) defines routes and authentication:

```hcl
resource "local_file" "api_spec" {
  filename = "${path.module}/api-spec.yaml"

  content = yamlencode({
    swagger = "2.0"
    info = {
      title   = "${var.service_name} API"
      version = "1.0"
    }
    host     = "${var.service_name}-gateway-${var.project_id}.uc.gateway.dev"
    schemes  = ["https"]
    produces = ["application/json"]

    # Firebase/Google Authentication
    securityDefinitions = {
      google_id_token = {
        authorizationUrl = ""
        flow             = "implicit"
        type             = "oauth2"
        "x-google-issuer"    = "https://accounts.google.com"
        "x-google-jwks_uri"  = "https://www.googleapis.com/oauth2/v3/certs"
        "x-google-audiences" = var.api_audience
      }
    }

    paths = {
      "/users" = {
        get = {
          summary     = "List users"
          operationId = "listUsers"
          security    = [{ google_id_token = [] }]
          "x-google-backend" = {
            address = "${google_cloud_run_v2_service.api.uri}/users"
          }
          responses = {
            "200" = { description = "Success" }
          }
        }
      }
    }
  })
}
```

## Service Account for Gateway

```hcl
resource "google_service_account" "api_gateway" {
  account_id   = "api-gateway-sa"
  display_name = "API Gateway Service Account"
  description  = "Service account for API Gateway to invoke Cloud Run"
}

# Allow API Gateway to invoke Cloud Run services
resource "google_cloud_run_service_iam_member" "gateway_invoker" {
  location = var.region
  service  = google_cloud_run_v2_service.api.name
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.api_gateway.email}"
}
```

## Multiple Gateway Environments

```hcl
locals {
  environments = {
    staging    = { min_instances = 0, max_instances = 5 }
    production = { min_instances = 2, max_instances = 100 }
  }
}

resource "google_api_gateway_gateway" "envs" {
  for_each = local.environments

  provider   = google-beta
  api_config = google_api_gateway_api_config.main.id
  gateway_id = "${var.service_name}-${each.key}-gateway"
  region     = var.region
  project    = var.project_id

  display_name = "${var.service_name} API Gateway (${each.key})"

  labels = {
    environment = each.key
  }
}
```

## API Gateway Monitoring

```hcl
resource "google_monitoring_alert_policy" "gateway_error_rate" {
  display_name = "API Gateway High Error Rate"
  combiner     = "OR"

  conditions {
    display_name = "4xx/5xx error rate above 5%"
    condition_threshold {
      filter = <<-EOT
        resource.type="apigateway.googleapis.com/Gateway"
        AND metric.type="apigateway.googleapis.com/http/response_count"
        AND metric.labels.response_code_class != "2xx"
      EOT
      comparison      = "COMPARISON_GT"
      threshold_value = 10
      duration        = "60s"
      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_RATE"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.name]
}
```

## Conclusion

GCP API Gateway in OpenTofu provides a managed gateway for Cloud Run and Cloud Functions backends. Define your API contract in OpenAPI 2.0 specs, use Firebase or Google JWT authentication for securing endpoints, and create separate gateway deployments per environment. The gateway_config backend_config allows using a service account to invoke Cloud Run services securely without public access, keeping your backend services private while the API Gateway handles public traffic.
