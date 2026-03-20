# How to Deploy Grafana with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Grafana, Kubernetes, Helm, Monitoring, Infrastructure as Code, Dashboards

Description: Learn how to deploy Grafana to Kubernetes using OpenTofu with pre-configured data sources, dashboards, and persistent storage for production-ready visualization.

---

Grafana is the leading open-source analytics and visualization platform used alongside Prometheus, Loki, and other data sources. Deploying Grafana via OpenTofu ensures your dashboard configuration, data sources, and LDAP/SSO settings are version-controlled and reproducible.

## Deploying Grafana with the Helm Provider

```hcl
# main.tf
terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.24"
    }
  }
}

provider "helm" {
  kubernetes {
    host                   = var.cluster_endpoint
    cluster_ca_certificate = base64decode(var.cluster_ca_cert)
    token                  = var.cluster_token
  }
}

provider "kubernetes" {
  host                   = var.cluster_endpoint
  cluster_ca_certificate = base64decode(var.cluster_ca_cert)
  token                  = var.cluster_token
}

resource "kubernetes_namespace" "monitoring" {
  metadata {
    name = "monitoring"
  }
}

resource "helm_release" "grafana" {
  name       = "grafana"
  repository = "https://grafana.github.io/helm-charts"
  chart      = "grafana"
  version    = "7.2.3"
  namespace  = kubernetes_namespace.monitoring.metadata[0].name

  wait    = true
  timeout = 300

  values = [
    yamlencode({
      # Set admin credentials
      adminUser     = "admin"
      adminPassword = var.grafana_admin_password

      # Enable persistent storage
      persistence = {
        enabled          = true
        storageClassName = var.storage_class
        size             = "10Gi"
      }

      # Resource limits
      resources = {
        requests = {
          cpu    = "100m"
          memory = "256Mi"
        }
        limits = {
          cpu    = "500m"
          memory = "512Mi"
        }
      }

      # Configure data sources via provisioning
      datasources = {
        "datasources.yaml" = {
          apiVersion = 1
          datasources = [
            {
              name      = "Prometheus"
              type      = "prometheus"
              url       = "http://kube-prometheus-stack-prometheus:9090"
              isDefault = true
              access    = "proxy"
            },
            {
              name   = "Loki"
              type   = "loki"
              url    = "http://loki:3100"
              access = "proxy"
            }
          ]
        }
      }

      # Configure dashboard providers
      dashboardProviders = {
        "dashboardproviders.yaml" = {
          apiVersion = 1
          providers = [
            {
              name            = "default"
              orgId           = 1
              folder          = ""
              type            = "file"
              disableDeletion = false
              options = {
                path = "/var/lib/grafana/dashboards/default"
              }
            }
          ]
        }
      }

      # Load community dashboards by ID
      dashboards = {
        default = {
          # Kubernetes cluster overview
          kubernetes-cluster = {
            gnetId     = 7249
            revision   = 1
            datasource = "Prometheus"
          }
          # Node Exporter full dashboard
          node-exporter = {
            gnetId     = 1860
            revision   = 37
            datasource = "Prometheus"
          }
        }
      }
    })
  ]
}
```

## Exposing Grafana with an Ingress

```hcl
# ingress.tf
resource "kubernetes_ingress_v1" "grafana" {
  metadata {
    name      = "grafana"
    namespace = kubernetes_namespace.monitoring.metadata[0].name
    annotations = {
      "kubernetes.io/ingress.class"                = "nginx"
      "cert-manager.io/cluster-issuer"             = "letsencrypt-prod"
      "nginx.ingress.kubernetes.io/ssl-redirect"   = "true"
    }
  }

  spec {
    tls {
      hosts       = [var.grafana_hostname]
      secret_name = "grafana-tls"
    }

    rule {
      host = var.grafana_hostname
      http {
        path {
          path      = "/"
          path_type = "Prefix"
          backend {
            service {
              name = "grafana"
              port {
                number = 80
              }
            }
          }
        }
      }
    }
  }
}
```

## Configuring SMTP for Alert Notifications

```hcl
# smtp_config.tf
resource "kubernetes_secret" "grafana_smtp" {
  metadata {
    name      = "grafana-smtp-secret"
    namespace = kubernetes_namespace.monitoring.metadata[0].name
  }

  data = {
    GF_SMTP_HOST     = var.smtp_host
    GF_SMTP_USER     = var.smtp_user
    GF_SMTP_PASSWORD = var.smtp_password
  }
}

# Reference the secret in the Helm release
# Add to the values block in the helm_release resource:
# envFromSecret: "grafana-smtp-secret"
```

## Best Practices

- Store the admin password in a secrets manager (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager) and reference it via a Kubernetes secret rather than embedding it in Terraform variables.
- Use Grafana provisioning (datasources, dashboards, alerting) rather than manual UI configuration so everything is code.
- Pin the Helm chart version to avoid unexpected upgrades that might change the admin API or dashboard schema.
- Enable `disableDeletion = false` for dashboard providers during initial setup — set to `true` only after stabilizing your dashboard set.
- Back up Grafana's SQLite database (or use PostgreSQL for HA deployments) before upgrading major versions.
