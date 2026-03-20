# How to Deploy MetalLB on Kubernetes with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, MetalLB, Load Balancer, OpenTofu, Helm, Bare Metal, BGP

Description: Learn how to deploy MetalLB on Kubernetes using OpenTofu and Helm to provide LoadBalancer services for bare metal and on-premises clusters using Layer 2 or BGP mode.

## Overview

MetalLB provides a network load balancer implementation for bare metal Kubernetes clusters where cloud provider load balancers are unavailable. OpenTofu deploys MetalLB via Helm and configures IP address pools and advertisement modes.

## Step 1: Deploy MetalLB with Helm

```hcl
# main.tf - Deploy MetalLB via Helm
resource "helm_release" "metallb" {
  name             = "metallb"
  repository       = "https://metallb.github.io/metallb"
  chart            = "metallb"
  version          = "0.14.3"
  namespace        = "metallb-system"
  create_namespace = true

  values = [yamlencode({
    controller = {
      resources = {
        requests = { cpu = "100m", memory = "100Mi" }
        limits   = { cpu = "300m", memory = "150Mi" }
      }
    }

    speaker = {
      # DaemonSet - runs on all nodes
      resources = {
        requests = { cpu = "100m", memory = "100Mi" }
        limits   = { cpu = "300m", memory = "150Mi" }
      }

      # Enable FRR (Free Range Routing) mode for BGP
      frr = {
        enabled = true
      }
    }
  })]
}
```

## Step 2: Configure Layer 2 Mode (Simple Setup)

```hcl
# Layer 2 mode - simplest setup, no BGP router required
resource "kubernetes_manifest" "ip_pool_l2" {
  depends_on = [helm_release.metallb]

  manifest = {
    apiVersion = "metallb.io/v1beta1"
    kind       = "IPAddressPool"
    metadata = {
      name      = "production-pool"
      namespace = "metallb-system"
    }
    spec = {
      addresses = [
        "192.168.10.100-192.168.10.150",  # Available IP range
        "192.168.10.200/28"               # CIDR notation
      ]
      autoAssign = true
    }
  }
}

# Advertise IPs via Layer 2
resource "kubernetes_manifest" "l2_advertisement" {
  depends_on = [kubernetes_manifest.ip_pool_l2]

  manifest = {
    apiVersion = "metallb.io/v1beta1"
    kind       = "L2Advertisement"
    metadata = {
      name      = "l2-advert"
      namespace = "metallb-system"
    }
    spec = {
      ipAddressPools = ["production-pool"]
      # Restrict to specific nodes
      nodeSelectors = [{
        matchLabels = {
          "node-role.kubernetes.io/worker" = "true"
        }
      }]
    }
  }
}
```

## Step 3: Configure BGP Mode (Production Setup)

```hcl
# BGP peer (your router/switch)
resource "kubernetes_manifest" "bgp_peer" {
  depends_on = [helm_release.metallb]

  manifest = {
    apiVersion = "metallb.io/v1beta2"
    kind       = "BGPPeer"
    metadata = {
      name      = "spine-router"
      namespace = "metallb-system"
    }
    spec = {
      myASN   = 64512  # Kubernetes cluster ASN
      peerASN = 64513  # Router ASN
      peerAddress = "10.0.0.1"  # Router IP
      holdTime    = "30s"
    }
  }
}

# BGP advertisement
resource "kubernetes_manifest" "bgp_advertisement" {
  manifest = {
    apiVersion = "metallb.io/v1beta1"
    kind       = "BGPAdvertisement"
    metadata = {
      name      = "bgp-advert"
      namespace = "metallb-system"
    }
    spec = {
      ipAddressPools = ["production-pool"]

      # Advertise with communities for traffic engineering
      communities = ["64512:100"]

      # Local preference for this cluster
      localPref = 100
    }
  }
}
```

## Step 4: Reserved IPs for Specific Services

```hcl
# Create a dedicated pool for specific services
resource "kubernetes_manifest" "ip_pool_reserved" {
  manifest = {
    apiVersion = "metallb.io/v1beta1"
    kind       = "IPAddressPool"
    metadata = {
      name      = "ingress-pool"
      namespace = "metallb-system"
    }
    spec = {
      addresses  = ["192.168.10.100/32"]  # Single IP for ingress
      autoAssign = false  # Only assign when explicitly requested
    }
  }
}

# Service requesting a specific IP from the reserved pool
resource "kubernetes_service" "ingress_lb" {
  metadata {
    name      = "ingress-nginx-controller"
    namespace = "ingress-nginx"
    annotations = {
      "metallb.universe.tf/address-pool"  = "ingress-pool"
      "metallb.universe.tf/loadBalancerIPs" = "192.168.10.100"
    }
  }

  spec {
    type = "LoadBalancer"
    selector = { "app.kubernetes.io/name" = "ingress-nginx" }
    port {
      name        = "http"
      port        = 80
      target_port = "http"
    }
    port {
      name        = "https"
      port        = 443
      target_port = "https"
    }
  }
}
```

## Summary

MetalLB deployed with OpenTofu brings cloud-like LoadBalancer functionality to bare metal and on-premises Kubernetes clusters. Layer 2 mode works with any network setup without BGP router configuration but is limited to single-node traffic routing. BGP mode provides true load distribution across nodes and integrates with enterprise network infrastructure for production deployments.
