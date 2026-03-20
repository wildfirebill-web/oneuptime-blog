# How to Create GCP IoT Core Registries with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, IoT Core, IoT, Device Registry, Infrastructure as Code

Description: Learn how to create GCP IoT Core device registries, configure Pub/Sub event notifications, and register IoT devices using OpenTofu.

## Introduction

GCP Cloud IoT Core provides a managed service for securely connecting and managing IoT devices. Device registries hold device credentials and route telemetry to Pub/Sub topics. OpenTofu manages registries, devices, and associated Pub/Sub topics as code.

> **Note:** Cloud IoT Core was deprecated by Google and retired in August 2023. This post covers the legacy API for historical reference; for new projects consider using GCP Pub/Sub directly or third-party IoT platforms.

## Enabling the API

```hcl
resource "google_project_service" "cloudiot" {
  project = var.project_id
  service = "cloudiot.googleapis.com"
}
```

## Pub/Sub Topics for Device Events

```hcl
resource "google_pubsub_topic" "telemetry" {
  name    = "${var.app_name}-telemetry"
  project = var.project_id
}

resource "google_pubsub_topic" "state" {
  name    = "${var.app_name}-device-state"
  project = var.project_id
}

# Grant IoT Core permission to publish to Pub/Sub

resource "google_pubsub_topic_iam_binding" "iot_publisher" {
  topic   = google_pubsub_topic.telemetry.id
  project = var.project_id
  role    = "roles/pubsub.publisher"
  members = ["serviceAccount:cloud-iot@system.gserviceaccount.com"]
}
```

## Creating a Device Registry

```hcl
resource "google_cloudiot_registry" "sensors" {
  name    = "${var.app_name}-sensors"
  project = var.project_id
  region  = var.region

  # Route device telemetry to Pub/Sub
  event_notification_configs {
    pubsub_topic_name = google_pubsub_topic.telemetry.id
  }

  # Route device state updates
  state_notification_config = {
    pubsub_topic_name = google_pubsub_topic.state.id
  }

  # HTTP and MQTT protocol support
  http_config = {
    http_enabled_state = "HTTP_ENABLED"
  }

  mqtt_config = {
    mqtt_enabled_state = "MQTT_ENABLED"
  }

  log_level = "INFO"

  credentials {
    public_key_certificate {
      format      = "X509_CERTIFICATE_PEM"
      certificate = file("${path.module}/certs/registry-ca.pem")
    }
  }
}
```

## Registering a Device

```hcl
resource "google_cloudiot_device" "sensor_001" {
  name     = "sensor-001"
  registry = google_cloudiot_registry.sensors.id

  credentials {
    public_key {
      format = "RSA_X509_PEM"
      key    = file("${path.module}/certs/device-001-public.pem")
    }
  }

  blocked = false  # set true to disable device

  metadata = {
    location = "floor-3-room-301"
    firmware = "v2.1.0"
  }

  gateway_config {
    gateway_type = "NON_GATEWAY"
  }
}
```

## Outputs

```hcl
output "registry_name" {
  value = google_cloudiot_registry.sensors.name
}

output "telemetry_topic" {
  value = google_pubsub_topic.telemetry.id
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

GCP Cloud IoT Core registries provide device management with secure MQTT/HTTP connections and automatic Pub/Sub routing. OpenTofu manages registries, Pub/Sub topics, IAM bindings, and device registrations as code - creating reproducible IoT infrastructure.
