# How to Deploy CoreDNS on EKS with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, EKS, CoreDNS, DNS, Kubernetes, Service Discovery, Infrastructure as Code

Description: Learn how to configure and tune CoreDNS on EKS using OpenTofu to optimize DNS resolution performance and configure custom DNS rules for service discovery.

## Introduction

CoreDNS is the default DNS server for Kubernetes clusters, handling service discovery and external DNS resolution for pods. On EKS, it runs as a managed add-on. This guide covers deploying, tuning, and customizing CoreDNS configuration for production workloads.

## Prerequisites

- OpenTofu v1.6+
- An existing EKS cluster
- Helm provider and Kubernetes provider configured

## Step 1: Deploy CoreDNS as an EKS Add-On

```hcl
# Deploy CoreDNS via the EKS managed add-on
resource "aws_eks_addon" "coredns" {
  cluster_name             = var.cluster_name
  addon_name               = "coredns"
  addon_version            = data.aws_eks_addon_version.coredns.version
  resolve_conflicts_on_update = "OVERWRITE"

  tags = {
    Name    = "coredns"
    Cluster = var.cluster_name
  }
}
```

## Step 2: Scale CoreDNS for Production

```hcl
# CoreDNS deployment patch to scale for production traffic
resource "kubernetes_deployment" "coredns_patch" {
  metadata {
    name      = "coredns"
    namespace = "kube-system"
  }

  spec {
    # Scale to at least 2 replicas for HA, more for large clusters
    replicas = max(2, ceil(var.expected_node_count / 10))

    selector {
      match_labels = { "k8s-app" = "kube-dns" }
    }

    template {
      metadata {
        labels = { "k8s-app" = "kube-dns" }
      }

      spec {
        container {
          name = "coredns"

          resources {
            requests = {
              cpu    = "100m"
              memory = "70Mi"
            }
            limits = {
              memory = "170Mi"
            }
          }
        }
      }
    }
  }
}
```

## Step 3: Customize CoreDNS Configuration

```hcl
# Update the CoreDNS ConfigMap with custom configuration
resource "kubernetes_config_map" "coredns" {
  metadata {
    name      = "coredns"
    namespace = "kube-system"
  }

  data = {
    Corefile = <<-EOF
      .:53 {
          errors
          health {
             lameduck 5s
          }
          ready
          kubernetes cluster.local in-addr.arpa ip6.arpa {
             pods insecure
             fallthrough in-addr.arpa ip6.arpa
             ttl 30
          }
          # Forward on-premises domain to corporate DNS
          forward corp.example.com 192.168.1.10 192.168.1.11 {
             force_tcp
          }
          # Cache DNS responses for better performance
          cache 30
          loop
          reload
          loadbalance
          # Enable Prometheus metrics
          prometheus :9153
          forward . /etc/resolv.conf {
             max_concurrent 1000
          }
          log . {
             class error
          }
      }
    EOF
  }
}
```

## Step 4: Configure HPA for CoreDNS

```hcl
# Horizontal Pod Autoscaler for CoreDNS based on CPU
resource "kubernetes_horizontal_pod_autoscaler_v2" "coredns" {
  metadata {
    name      = "coredns"
    namespace = "kube-system"
  }

  spec {
    scale_target_ref {
      api_version = "apps/v1"
      kind        = "Deployment"
      name        = "coredns"
    }

    min_replicas = 2
    max_replicas = 10

    metric {
      type = "Resource"
      resource {
        name = "cpu"
        target {
          type                = "Utilization"
          average_utilization = 70
        }
      }
    }
  }
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# Verify CoreDNS is running
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=20

# Test DNS resolution from a pod
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
```

## Conclusion

Properly tuned CoreDNS is critical for cluster-wide service discovery performance. Scale CoreDNS replicas with cluster size, enable caching to reduce upstream DNS load, and configure custom forwarding rules for hybrid environments. Monitor CoreDNS metrics via Prometheus (`coredns_dns_requests_total`) to detect DNS bottlenecks before they impact application performance.
