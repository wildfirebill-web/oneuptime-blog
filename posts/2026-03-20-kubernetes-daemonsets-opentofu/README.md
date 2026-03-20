# How to Create Kubernetes DaemonSets with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, DaemonSets, OpenTofu, Infrastructure, Monitoring, Logging

Description: Learn how to create Kubernetes DaemonSets with OpenTofu to ensure monitoring agents, log collectors, and node configuration tools run on every node in your cluster.

## Overview

DaemonSets ensure that a pod runs on all (or selected) nodes in a cluster. They're ideal for node-level infrastructure like log collectors (Fluentd), monitoring agents (Prometheus Node Exporter), and network plugins. OpenTofu manages DaemonSet definitions with node selectors and tolerations.

## Step 1: Basic DaemonSet for Log Collection

```hcl
# main.tf - Fluentd DaemonSet for log collection
resource "kubernetes_daemon_set_v1" "fluentd" {
  metadata {
    name      = "fluentd-log-collector"
    namespace = "kube-system"
    labels = {
      app = "fluentd"
    }
  }

  spec {
    selector {
      match_labels = {
        app = "fluentd"
      }
    }

    template {
      metadata {
        labels = {
          app = "fluentd"
        }
      }

      spec {
        service_account_name = kubernetes_service_account.fluentd_sa.metadata[0].name
        # Allow scheduling on master nodes
        toleration {
          key    = "node-role.kubernetes.io/control-plane"
          effect = "NoSchedule"
        }

        container {
          name  = "fluentd"
          image = "fluent/fluentd-kubernetes-daemonset:v1.16-debian-cloudwatch-1"

          env {
            name = "NODE_NAME"
            value_from {
              field_ref {
                field_path = "spec.nodeName"
              }
            }
          }

          resources {
            limits = {
              memory = "256Mi"
              cpu    = "200m"
            }
            requests = {
              cpu    = "100m"
              memory = "128Mi"
            }
          }

          # Mount host logs directory
          volume_mount {
            name       = "varlog"
            mount_path = "/var/log"
          }

          volume_mount {
            name       = "containers"
            mount_path = "/var/lib/docker/containers"
            read_only  = true
          }
        }

        volume {
          name = "varlog"
          host_path {
            path = "/var/log"
          }
        }

        volume {
          name = "containers"
          host_path {
            path = "/var/lib/docker/containers"
          }
        }
      }
    }

    update_strategy {
      type = "RollingUpdate"
      rolling_update {
        max_unavailable = "1"
      }
    }
  }
}
```

## Step 2: DaemonSet with Node Selector

```hcl
# DaemonSet running only on specific nodes (e.g., GPU nodes)
resource "kubernetes_daemon_set_v1" "nvidia_plugin" {
  metadata {
    name      = "nvidia-device-plugin"
    namespace = "kube-system"
  }

  spec {
    selector {
      match_labels = {
        app = "nvidia-device-plugin"
      }
    }

    template {
      metadata {
        labels = {
          app = "nvidia-device-plugin"
        }
      }

      spec {
        # Only run on nodes with NVIDIA GPUs
        node_selector = {
          "cloud.google.com/gke-accelerator" = "nvidia-tesla-t4"
        }

        toleration {
          key      = "nvidia.com/gpu"
          operator = "Exists"
          effect   = "NoSchedule"
        }

        container {
          name  = "nvidia-device-plugin"
          image = "nvcr.io/nvidia/k8s-device-plugin:v0.14.1"

          security_context {
            allow_privilege_escalation = false
            capabilities {
              drop = ["ALL"]
            }
          }
        }
      }
    }
  }
}
```

## Summary

Kubernetes DaemonSets with OpenTofu ensure critical node-level infrastructure runs on every node. Use tolerations to extend DaemonSets to control plane nodes, node selectors to limit them to specific node types, and rolling updates for zero-downtime deployments of monitoring agents and log collectors.
