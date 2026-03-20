# How to Build a Microservices Architecture with OpenTofu on GCP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, Microservices, Architecture, OpenTofu, GKE, Cloud Endpoints, Pub/Sub

Description: Learn how to build a production-ready microservices architecture on GCP using OpenTofu with GKE, Cloud Endpoints, Pub/Sub event messaging, and Workload Identity.

## Overview

Microservices on GCP use GKE for container orchestration, Cloud Endpoints or API Gateway for external access, Pub/Sub for event-driven communication, and Workload Identity to bind Kubernetes service accounts to GCP service accounts.

## Step 1: GKE Cluster

```hcl
# main.tf - GKE cluster for microservices

resource "google_container_cluster" "microservices" {
  name     = "microservices-cluster"
  location = "us-central1"

  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.gke.name

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  networking_config {
    enable_intranode_visibility = true
  }

  gateway_api_config {
    channel = "CHANNEL_STANDARD"
  }

  release_channel {
    channel = "REGULAR"
  }

  ip_allocation_policy {
    cluster_secondary_range_name  = "pod-range"
    services_secondary_range_name = "service-range"
  }
}

resource "google_container_node_pool" "workloads" {
  name       = "workloads"
  cluster    = google_container_cluster.microservices.id
  node_count = 3

  autoscaling {
    min_node_count = 3
    max_node_count = 20
  }

  node_config {
    machine_type = "e2-standard-4"
    disk_type    = "pd-balanced"

    workload_metadata_config {
      mode = "GKE_METADATA"
    }
  }
}
```

## Step 2: Workload Identity for Each Service

```hcl
# GCP service account per microservice
resource "google_service_account" "order_service" {
  account_id   = "order-service"
  display_name = "Order Service"
}

# Bind K8s service account to GCP service account
resource "google_service_account_iam_binding" "order_workload_identity" {
  service_account_id = google_service_account.order_service.name
  role               = "roles/iam.workloadIdentityUser"

  members = [
    "serviceAccount:${var.project_id}.svc.id.goog[orders/order-service]"
  ]
}

# Grant permissions to the service
resource "google_pubsub_topic_iam_member" "order_service_publish" {
  topic  = google_pubsub_topic.order_events.id
  role   = "roles/pubsub.publisher"
  member = "serviceAccount:${google_service_account.order_service.email}"
}

# Kubernetes service account annotation
resource "kubernetes_service_account" "order_service" {
  metadata {
    name      = "order-service"
    namespace = "orders"
    annotations = {
      "iam.gke.io/gcp-service-account" = google_service_account.order_service.email
    }
  }
}
```

## Step 3: Pub/Sub for Event-Driven Microservices

```hcl
# Pub/Sub topics for domain events
resource "google_pubsub_topic" "order_events" {
  name = "order-events"

  message_retention_duration = "604800s"  # 7 days
}

resource "google_pubsub_subscription" "payment_service" {
  name  = "payment-service-orders"
  topic = google_pubsub_topic.order_events.id

  ack_deadline_seconds    = 30
  message_retention_duration = "604800s"

  retry_policy {
    minimum_backoff = "10s"
    maximum_backoff = "600s"
  }

  dead_letter_policy {
    dead_letter_topic     = google_pubsub_topic.dead_letter.id
    max_delivery_attempts = 5
  }
}

resource "google_pubsub_subscription_iam_member" "payment_consume" {
  subscription = google_pubsub_subscription.payment_service.name
  role         = "roles/pubsub.subscriber"
  member       = "serviceAccount:${google_service_account.payment_service.email}"
}
```

## Step 4: Cloud Endpoints for API Gateway

```hcl
# Cloud Endpoints service for external API access
resource "google_endpoints_service" "api" {
  service_name   = "api.endpoints.${var.project_id}.cloud.goog"
  project        = var.project_id
  grpc_config    = file("${path.module}/api-config.yaml")
  protoc_output_base64 = filebase64("${path.module}/api_descriptor.pb")
}
```

## Summary

Microservices on GCP built with OpenTofu use GKE Workload Identity to bind Kubernetes service accounts to GCP service accounts, enabling each microservice to access only its authorized GCP resources without static credentials. Pub/Sub provides durable event delivery with dead-letter queues for failed messages. The GKE Gateway API with standard channel provides Kubernetes-native traffic management aligned with the upstream Gateway API specification.
