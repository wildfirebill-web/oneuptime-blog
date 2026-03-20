# How to Use the http Data Source in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Data Sources, http, HTTP, REST API, Infrastructure as Code, DevOps

Description: A guide to using the http data source in OpenTofu to fetch data from HTTP APIs and web endpoints during configuration.

## Introduction

The `http` data source from the http provider makes an HTTP GET request to a given URL and exposes the response body and headers as data source attributes. It is useful for fetching configuration data from APIs, checking remote resources, or retrieving content that doesn't have a dedicated provider.

## Setting Up the HTTP Provider

```hcl
terraform {
  required_providers {
    http = {
      source  = "hashicorp/http"
      version = "~> 3.0"
    }
  }
}
```

## Basic HTTP Request

```hcl
# Fetch data from a public API
data "http" "ip_info" {
  url = "https://api.ipify.org?format=json"
}

output "public_ip" {
  value = jsondecode(data.http.ip_info.response_body)["ip"]
}
```

## Fetching GitHub Release Information

```hcl
# Get the latest release of a GitHub repository
data "http" "github_release" {
  url = "https://api.github.com/repos/opentofu/opentofu/releases/latest"

  request_headers = {
    Accept = "application/vnd.github.v3+json"
  }
}

locals {
  latest_version = jsondecode(data.http.github_release.response_body)["tag_name"]
}

output "latest_opentofu_version" {
  value = local.latest_version
}
```

## HTTP Request with Headers

```hcl
data "http" "internal_api" {
  url = "https://internal-api.company.com/config/app-settings"

  request_headers = {
    Authorization = "Bearer ${var.api_token}"
    Accept        = "application/json"
    X-API-Version = "v2"
  }
}

locals {
  app_settings = jsondecode(data.http.internal_api.response_body)
}
```

## Using Response Headers

```hcl
data "http" "check_endpoint" {
  url = "https://api.example.com/health"
}

output "response_info" {
  value = {
    status_code   = data.http.check_endpoint.status_code
    content_type  = data.http.check_endpoint.response_headers["Content-Type"]
    body          = data.http.check_endpoint.response_body
  }
}
```

## Fetching EC2 Metadata

```hcl
# Fetch instance metadata from AWS metadata service
data "http" "instance_identity" {
  url = "http://169.254.169.254/latest/dynamic/instance-identity/document"
}

locals {
  instance_identity = jsondecode(data.http.instance_identity.response_body)
  account_id        = local.instance_identity["accountId"]
  region            = local.instance_identity["region"]
  instance_type     = local.instance_identity["instanceType"]
}
```

## Conditional Response Handling

```hcl
data "http" "feature_flags" {
  url = "https://feature-flags.company.com/api/flags/${var.environment}"

  request_headers = {
    Authorization = "Bearer ${var.ff_token}"
  }
}

locals {
  flags = data.http.feature_flags.status_code == 200 ? (
    jsondecode(data.http.feature_flags.response_body)
  ) : {}

  enable_new_feature = lookup(local.flags, "new_feature", false)
}
```

## Adding Lifecycle Postcondition

```hcl
data "http" "config_api" {
  url = "https://config.company.com/api/v1/settings"

  request_headers = {
    Authorization = "Bearer ${var.config_token}"
  }

  lifecycle {
    postcondition {
      condition     = self.status_code == 200
      error_message = "Config API returned status ${self.status_code}. Expected 200."
    }
  }
}
```

## Retry Configuration

```hcl
data "http" "flaky_api" {
  url = "https://api.example.com/data"

  # Retry settings (provider version 3.4+)
  retry {
    attempts     = 3
    min_delay_ms = 1000
    max_delay_ms = 5000
  }
}
```

## Fetching Remote tfvars

```hcl
# Fetch centralized configuration from a config server
data "http" "remote_config" {
  url = "https://config.company.com/environments/${var.environment}/opentofu.json"

  request_headers = {
    Authorization = "Bearer ${var.config_token}"
  }
}

locals {
  remote_settings = jsondecode(data.http.remote_config.response_body)
  db_instance_class = local.remote_settings["db_instance_class"]
  instance_type     = local.remote_settings["instance_type"]
}
```

## Validating TLS Certificates

```hcl
data "http" "verify_endpoint" {
  url = "https://api.company.com/health"

  # By default, TLS certificate validation is enforced
  # Do NOT disable unless absolutely necessary
  # insecure = false  # This is the default
}
```

## Caching Considerations

```hcl
# HTTP data sources are fetched on every plan and apply
# For expensive or rate-limited APIs, consider:

# 1. Store results in SSM Parameter Store
resource "aws_ssm_parameter" "cached_config" {
  name  = "/myapp/remote-config"
  type  = "String"
  value = data.http.remote_config.response_body
}

# 2. Use a trigger to only refresh when needed
resource "terraform_data" "config_refresh" {
  triggers_replace = {
    # Only refresh on manual trigger or schedule
    refresh_date = var.config_refresh_date
  }
}
```

## Conclusion

The `http` data source is a versatile tool for fetching configuration data, checking external APIs, and retrieving remote content. It supports custom headers for authentication, response status validation through postconditions, and retry logic for unreliable endpoints. Since HTTP requests are made on every plan and apply, be mindful of rate limits and latency. For authenticated requests, use ephemeral variables for tokens to avoid storing credentials in state. The `http` data source works best for read-only configuration lookups from stable, low-latency APIs.
