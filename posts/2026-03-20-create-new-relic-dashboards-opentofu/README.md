# How to Create New Relic Dashboards with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, New Relic, Dashboards, Observability, Infrastructure as Code

Description: Learn how to create New Relic dashboards and visualizations with OpenTofu to codify your observability views alongside your infrastructure.

New Relic dashboards provide visibility into application performance, infrastructure health, and business metrics. Managing dashboards as code with OpenTofu ensures consistent views across teams and prevents configuration drift.

## Provider Configuration

```hcl
terraform {
  required_providers {
    newrelic = {
      source  = "newrelic/newrelic"
      version = "~> 3.0"
    }
  }
}

provider "newrelic" {
  account_id = var.newrelic_account_id
  api_key    = var.newrelic_api_key
  region     = "US"
}
```

## Creating a Dashboard

```hcl
resource "newrelic_one_dashboard" "application" {
  name        = "Application Overview"
  description = "Key metrics for the production application"
  permissions = "public_read_write"  # or "public_read_only", "private"

  page {
    name = "Overview"

    # Error rate widget
    widget_line {
      title  = "Error Rate (%)"
      row    = 1
      column = 1
      width  = 6
      height = 3

      nrql_query {
        account_id = var.newrelic_account_id
        query      = "SELECT percentage(count(*), WHERE error IS true) FROM Transaction WHERE appName = 'myapp' TIMESERIES AUTO"
      }
    }

    # Throughput widget
    widget_line {
      title  = "Throughput (rpm)"
      row    = 1
      column = 7
      width  = 6
      height = 3

      nrql_query {
        account_id = var.newrelic_account_id
        query      = "SELECT rate(count(*), 1 minute) FROM Transaction WHERE appName = 'myapp' TIMESERIES AUTO"
      }
    }

    # P95 response time
    widget_area {
      title  = "P95 Response Time"
      row    = 4
      column = 1
      width  = 8
      height = 3

      nrql_query {
        account_id = var.newrelic_account_id
        query      = "SELECT percentile(duration, 95) FROM Transaction WHERE appName = 'myapp' TIMESERIES AUTO"
      }
    }

    # Apdex score
    widget_billboard {
      title  = "Apdex Score"
      row    = 4
      column = 9
      width  = 4
      height = 3

      nrql_query {
        account_id = var.newrelic_account_id
        query      = "SELECT apdex(duration, t:0.5) FROM Transaction WHERE appName = 'myapp'"
      }

      # Color thresholds
      critical = 0.7
      warning  = 0.85
    }
  }
}
```

## Adding a Second Page

```hcl
resource "newrelic_one_dashboard" "infrastructure" {
  name        = "Infrastructure Health"
  permissions = "public_read_only"

  page {
    name = "Hosts"

    widget_table {
      title  = "Host Summary"
      row    = 1
      column = 1
      width  = 12
      height = 4

      nrql_query {
        account_id = var.newrelic_account_id
        query      = "SELECT latest(cpuPercent), latest(memoryUsedPercent), latest(diskUsedPercent) FROM SystemSample FACET hostname LIMIT 50"
      }
    }

    widget_heatmap {
      title  = "CPU Heatmap"
      row    = 5
      column = 1
      width  = 12
      height = 3

      nrql_query {
        account_id = var.newrelic_account_id
        query      = "SELECT histogram(cpuPercent, 10, 10) FROM SystemSample FACET hostname TIMESERIES AUTO"
      }
    }
  }

  page {
    name = "Kubernetes"

    widget_line {
      title  = "Pod Restarts"
      row    = 1
      column = 1
      width  = 12
      height = 3

      nrql_query {
        account_id = var.newrelic_account_id
        query      = "SELECT sum(restartCount) FROM K8sContainerSample FACET podName TIMESERIES AUTO"
      }
    }
  }
}
```

## Dashboard Variables (Template Variables)

```hcl
resource "newrelic_one_dashboard" "multi_env" {
  name = "Multi-Environment Overview"

  variable {
    name         = "environment"
    title        = "Environment"
    type         = "nrql"
    default_values = ["production"]
    is_multi_selection = false

    nrql_query {
      account_ids = [var.newrelic_account_id]
      query       = "SELECT uniques(environment) FROM Transaction LIMIT 10"
    }
  }

  page {
    name = "Overview"

    widget_line {
      title  = "Error Rate by Environment"
      row    = 1
      column = 1
      width  = 12
      height = 3

      nrql_query {
        account_id = var.newrelic_account_id
        query      = "SELECT percentage(count(*), WHERE error IS true) FROM Transaction WHERE environment = '{{environment}}' TIMESERIES AUTO"
      }
    }
  }
}
```

## Conclusion

New Relic dashboards in OpenTofu give you version-controlled observability views. Define NRQL queries for all your key metrics, organize them into pages, and use template variables for dynamic filtering. Dashboard changes go through code review, preventing accidental deletion of critical views.
