# How to Configure the Datadog Provider in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Datadog, Infrastructure as Code, IaC, Datadog Provider, Monitoring

Description: Learn how to configure the Datadog provider in OpenTofu with API and app key authentication to manage monitors, dashboards, and SLOs as code.

## Introduction

The Datadog provider for OpenTofu lets you manage monitors, dashboards, SLOs, synthetic tests, and other Datadog resources as code. This guide covers configuring the provider, authenticating with API and application keys, and creating your first Datadog resources.

## Prerequisites

- OpenTofu v1.6+
- A Datadog account with API access
- Datadog API key and Application key

## Step 1: Install and Configure the Provider

```hcl
# versions.tf

terraform {
  required_version = ">= 1.6.0"
  required_providers {
    datadog = {
      source  = "DataDog/datadog"
      version = "~> 3.39"
    }
  }
}

# Configure the Datadog provider
provider "datadog" {
  api_key = var.datadog_api_key
  app_key = var.datadog_app_key
  api_url = "https://api.datadoghq.com/"  # Use api.datadoghq.eu for EU
}
```

## Step 2: Set Up Authentication

```bash
# Use environment variables for authentication (recommended)
export DD_API_KEY="your-datadog-api-key"
export DD_APP_KEY="your-datadog-app-key"
```

```hcl
# variables.tf
variable "datadog_api_key" {
  description = "Datadog API key"
  type        = string
  sensitive   = true
}

variable "datadog_app_key" {
  description = "Datadog Application key"
  type        = string
  sensitive   = true
}
```

When environment variables `DD_API_KEY` and `DD_APP_KEY` are set, the provider reads them automatically and no explicit variable values are needed.

## Step 3: Create a Monitor

```hcl
# monitors.tf
resource "datadog_monitor" "cpu_high" {
  name    = "High CPU Usage on ${var.environment}"
  type    = "metric alert"
  message = "CPU usage is above 90% on {{host.name}}. @pagerduty-alerts"

  query = "avg(last_5m):avg:system.cpu.user{env:${var.environment}} by {host} > 90"

  monitor_thresholds {
    critical = 90
    warning  = 80
  }

  notify_no_data    = false
  renotify_interval = 60

  tags = ["env:${var.environment}", "team:platform", "managed-by:opentofu"]
}
```

## Step 4: Create a Dashboard

```hcl
# dashboards.tf
resource "datadog_dashboard" "service_overview" {
  title       = "${var.environment} Service Overview"
  description = "Key metrics for ${var.environment} environment"
  layout_type = "ordered"

  widget {
    timeseries_definition {
      title = "CPU Usage by Host"
      request {
        q            = "avg:system.cpu.user{env:${var.environment}} by {host}"
        display_type = "line"
      }
    }
  }

  widget {
    query_value_definition {
      title = "P99 Latency"
      request {
        q          = "p99:trace.web.request{env:${var.environment}}"
        aggregator = "last"
      }
    }
  }
}
```

## Step 5: Define Outputs

```hcl
# outputs.tf
output "monitor_id" {
  description = "The ID of the CPU monitor"
  value       = datadog_monitor.cpu_high.id
}

output "dashboard_url" {
  description = "URL of the service overview dashboard"
  value       = datadog_dashboard.service_overview.url
}
```

## Step 6: Deploy

```bash
# Initialize OpenTofu and download the Datadog provider
tofu init

# Validate configuration syntax
tofu validate

# Preview planned changes
tofu plan -var="datadog_api_key=$DD_API_KEY" -var="datadog_app_key=$DD_APP_KEY"

# Apply configuration
tofu apply
```

## Common Issues and Solutions

### Authentication Errors
Verify the API key is an API key (not an App key) and the App key has the correct permissions (e.g., `monitors_write`, `dashboards_write`). Check for trailing spaces in environment variables.

### Rate Limiting
Datadog enforces API rate limits. Add `depends_on` to serialize resource creation, or use `-parallelism=5` with `tofu apply` to reduce concurrent API calls.

### Provider Version Conflicts
Pin to a specific provider version range (e.g., `~> 3.39`) to ensure reproducible deployments. Breaking changes in the Datadog provider are tracked in the [provider changelog](https://github.com/DataDog/terraform-provider-datadog/blob/master/CHANGELOG.md).

## Conclusion

The Datadog provider (`DataDog/datadog`) lets you manage monitors, dashboards, SLOs, and synthetic tests as code. Authenticate with `DD_API_KEY` and `DD_APP_KEY` environment variables, or pass them as sensitive variables. Managing Datadog resources with OpenTofu ensures consistent alerting configurations across environments and enables code review for monitoring changes.
