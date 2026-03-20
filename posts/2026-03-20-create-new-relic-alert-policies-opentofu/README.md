# How to Create New Relic Alert Policies with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, New Relic, Alerting, Monitoring, Infrastructure as Code

Description: Learn how to create New Relic alert policies, conditions, and notification channels with OpenTofu to automate your observability configuration.

New Relic alert policies group related alert conditions and define how notifications are routed when incidents occur. Managing them in OpenTofu ensures consistent alerting across services and environments.

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
  region     = "US"  # or "EU"
}
```

## Creating an Alert Policy

```hcl
resource "newrelic_alert_policy" "application" {
  name                = "Application Alerts"
  incident_preference = "PER_CONDITION_AND_TARGET"
  # PER_POLICY: one incident per policy
  # PER_CONDITION: one incident per condition
  # PER_CONDITION_AND_TARGET: one incident per condition per entity
}
```

## NRQL Alert Conditions

```hcl
# High error rate condition
resource "newrelic_nrql_alert_condition" "error_rate" {
  policy_id   = newrelic_alert_policy.application.id
  name        = "High Error Rate"
  type        = "static"
  description = "Alert when error rate exceeds 5%"
  enabled     = true

  nrql {
    query = "SELECT percentage(count(*), WHERE error IS true) FROM Transaction WHERE appName = 'myapp'"
  }

  critical {
    operator              = "above"
    threshold             = 5
    threshold_duration    = 300  # 5 minutes
    threshold_occurrences = "ALL"
  }

  warning {
    operator              = "above"
    threshold             = 2
    threshold_duration    = 300
    threshold_occurrences = "ALL"
  }

  # Evaluate every minute, look back 1 minute
  aggregation_window  = 60
  aggregation_method  = "event_flow"
  aggregation_delay   = 120
}

# High response time condition
resource "newrelic_nrql_alert_condition" "response_time" {
  policy_id = newrelic_alert_policy.application.id
  name      = "High Response Time"
  type      = "static"
  enabled   = true

  nrql {
    query = "SELECT percentile(duration, 95) FROM Transaction WHERE appName = 'myapp'"
  }

  critical {
    operator              = "above"
    threshold             = 2.0  # 2 seconds
    threshold_duration    = 300
    threshold_occurrences = "ALL"
  }

  aggregation_window = 60
  aggregation_method = "event_flow"
  aggregation_delay  = 120
}
```

## Notification Channels

```hcl
# Email notification
resource "newrelic_alert_channel" "email" {
  name = "Engineering Team Email"
  type = "email"

  config {
    recipients              = "engineering@example.com"
    include_json_attachment = "1"
  }
}

# Slack notification
resource "newrelic_alert_channel" "slack" {
  name = "Alerts Slack Channel"
  type = "slack"

  config {
    url     = var.slack_webhook_url
    channel = "#alerts"
  }
}

# PagerDuty integration
resource "newrelic_alert_channel" "pagerduty" {
  name = "PagerDuty Oncall"
  type = "pagerduty"

  config {
    service_key = var.pagerduty_service_key
  }
}
```

## Connecting Channels to Policy

```hcl
resource "newrelic_alert_policy_channel" "application_channels" {
  policy_id   = newrelic_alert_policy.application.id
  channel_ids = [
    newrelic_alert_channel.email.id,
    newrelic_alert_channel.slack.id,
    newrelic_alert_channel.pagerduty.id,
  ]
}
```

## Multiple Policies with for_each

```hcl
locals {
  teams = ["frontend", "backend", "data"]
}

resource "newrelic_alert_policy" "teams" {
  for_each            = toset(local.teams)
  name                = "${each.key} Alerts"
  incident_preference = "PER_CONDITION_AND_TARGET"
}

resource "newrelic_nrql_alert_condition" "team_error_rate" {
  for_each  = toset(local.teams)
  policy_id = newrelic_alert_policy.teams[each.key].id
  name      = "${each.key} Error Rate"
  type      = "static"
  enabled   = true

  nrql {
    query = "SELECT percentage(count(*), WHERE error IS true) FROM Transaction WHERE team = '${each.key}'"
  }

  critical {
    operator              = "above"
    threshold             = 5
    threshold_duration    = 300
    threshold_occurrences = "ALL"
  }

  aggregation_window = 60
  aggregation_method = "event_flow"
  aggregation_delay  = 120
}
```

## Conclusion

New Relic alert policies in OpenTofu give you version-controlled, consistent alerting. Define NRQL conditions for precise metric-based alerts, configure notification channels for routing, and use for_each to apply the same alert patterns across multiple services or teams without duplicating configuration.
