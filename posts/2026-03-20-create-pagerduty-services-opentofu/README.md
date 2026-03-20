# How to Create PagerDuty Services and Escalation Policies with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, PagerDuty, Incident Management, On-Call, Infrastructure as Code

Description: Learn how to create PagerDuty services, escalation policies, and schedules with OpenTofu to automate your incident response configuration.

PagerDuty services, escalation policies, and on-call schedules define how incidents are routed and who gets paged. Managing this configuration in OpenTofu ensures consistent incident response across teams and makes changes auditable.

## Provider Configuration

```hcl
terraform {
  required_providers {
    pagerduty = {
      source  = "PagerDuty/pagerduty"
      version = "~> 3.0"
    }
  }
}

provider "pagerduty" {
  token = var.pagerduty_token
}
```

## Creating an On-Call Schedule

```hcl
# Look up existing users

data "pagerduty_user" "alice" {
  email = "alice@example.com"
}

data "pagerduty_user" "bob" {
  email = "bob@example.com"
}

resource "pagerduty_schedule" "backend_oncall" {
  name      = "Backend On-Call"
  time_zone = "America/New_York"

  layer {
    name                         = "Primary"
    start                        = "2024-01-01T00:00:00-05:00"
    rotation_virtual_start       = "2024-01-01T00:00:00-05:00"
    rotation_turn_length_seconds = 604800  # 1 week

    users = [
      data.pagerduty_user.alice.id,
      data.pagerduty_user.bob.id,
    ]

    restriction {
      type              = "weekly_restriction"
      start_time_of_day = "09:00:00"
      start_day_of_week = 1  # Monday
      duration_seconds  = 43200  # 12 hours
    }
  }
}
```

## Creating an Escalation Policy

```hcl
resource "pagerduty_escalation_policy" "backend" {
  name      = "Backend Escalation Policy"
  num_loops = 2  # Loop twice before stopping

  rule {
    escalation_delay_in_minutes = 30  # Wait 30 min before escalating

    target {
      type = "schedule_reference"
      id   = pagerduty_schedule.backend_oncall.id
    }
  }

  rule {
    escalation_delay_in_minutes = 30

    target {
      type = "user_reference"
      id   = data.pagerduty_user.alice.id  # Engineering manager
    }
  }
}
```

## Creating a Service

```hcl
resource "pagerduty_service" "api" {
  name                    = "API Service"
  description             = "Production API service"
  escalation_policy       = pagerduty_escalation_policy.backend.id
  auto_resolve_timeout    = 14400   # 4 hours
  acknowledgement_timeout = 600     # 10 minutes

  incident_urgency_rule {
    type    = "constant"
    urgency = "high"
  }

  alert_creation = "create_alerts_and_incidents"
}

# Service integration (for Datadog, CloudWatch, etc.)
resource "pagerduty_service_integration" "datadog" {
  name    = "Datadog"
  service = pagerduty_service.api.id
  vendor  = data.pagerduty_vendor.datadog.id
}

data "pagerduty_vendor" "datadog" {
  name = "Datadog"
}
```

## Response Plays

```hcl
resource "pagerduty_response_play" "database_incident" {
  name        = "Database Incident Response"
  from        = "oncall@example.com"
  description = "Standard response play for database incidents"

  responder {
    type = "escalation_policy_reference"
    id   = pagerduty_escalation_policy.backend.id
  }

  subscriber {
    type = "user_reference"
    id   = data.pagerduty_user.alice.id
  }

  conference_number = "+1-555-0100"
  conference_url    = "https://meet.example.com/db-incident"
}
```

## Multiple Services with for_each

```hcl
locals {
  services = {
    api       = { urgency = "high", timeout = 14400 }
    worker    = { urgency = "low",  timeout = 28800 }
    scheduler = { urgency = "low",  timeout = 28800 }
  }
}

resource "pagerduty_service" "services" {
  for_each = local.services

  name              = "${each.key} Service"
  escalation_policy = pagerduty_escalation_policy.backend.id
  auto_resolve_timeout    = each.value.timeout

  incident_urgency_rule {
    type    = "constant"
    urgency = each.value.urgency
  }
}
```

## Conclusion

PagerDuty services, escalation policies, and schedules managed in OpenTofu give you version-controlled incident response configuration. Define on-call rotations, multi-level escalations, and service-specific urgency rules. Changes to who gets paged go through code review, preventing misconfigured escalations from leaving incidents unacknowledged.
