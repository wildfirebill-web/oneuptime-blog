# How to Configure VPN Connections on GCP with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, VPN, Cloud VPN, Networking, Infrastructure as Code

Description: Learn how to provision Google Cloud VPN tunnels using OpenTofu to connect GCP VPC networks with on-premises networks or other clouds securely.

## Introduction

Google Cloud VPN creates IPsec VPN tunnels between your GCP VPC and on-premises or other cloud networks. OpenTofu manages the full stack - from VPN gateways to forwarding rules and tunnels - as version-controlled code.

## Provider Configuration

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
}
```

## VPC Network

```hcl
resource "google_compute_network" "main" {
  name                    = "main-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "main" {
  name          = "main-subnet"
  ip_cidr_range = "10.0.0.0/16"
  region        = var.region
  network       = google_compute_network.main.id
}
```

## VPN Gateway

```hcl
resource "google_compute_vpn_gateway" "main" {
  name    = "main-vpn-gateway"
  network = google_compute_network.main.id
  region  = var.region
}
```

## Static IP and Forwarding Rules

```hcl
resource "google_compute_address" "vpn_static_ip" {
  name   = "vpn-static-ip"
  region = var.region
}

resource "google_compute_forwarding_rule" "esp" {
  name        = "vpn-rule-esp"
  ip_protocol = "ESP"
  ip_address  = google_compute_address.vpn_static_ip.address
  target      = google_compute_vpn_gateway.main.id
  region      = var.region
}

resource "google_compute_forwarding_rule" "udp500" {
  name        = "vpn-rule-udp500"
  ip_protocol = "UDP"
  port_range  = "500"
  ip_address  = google_compute_address.vpn_static_ip.address
  target      = google_compute_vpn_gateway.main.id
  region      = var.region
}

resource "google_compute_forwarding_rule" "udp4500" {
  name        = "vpn-rule-udp4500"
  ip_protocol = "UDP"
  port_range  = "4500"
  ip_address  = google_compute_address.vpn_static_ip.address
  target      = google_compute_vpn_gateway.main.id
  region      = var.region
}
```

## VPN Tunnel

```hcl
resource "google_compute_vpn_tunnel" "main" {
  name                    = "main-vpn-tunnel"
  peer_ip                 = var.on_prem_public_ip
  shared_secret           = var.vpn_shared_key
  target_vpn_gateway      = google_compute_vpn_gateway.main.id
  local_traffic_selector  = ["0.0.0.0/0"]
  remote_traffic_selector = [var.on_prem_cidr]

  depends_on = [
    google_compute_forwarding_rule.esp,
    google_compute_forwarding_rule.udp500,
    google_compute_forwarding_rule.udp4500,
  ]
}
```

## Routes Through the Tunnel

```hcl
resource "google_compute_route" "vpn_route" {
  name                = "vpn-route-to-on-prem"
  network             = google_compute_network.main.name
  dest_range          = var.on_prem_cidr
  next_hop_vpn_tunnel = google_compute_vpn_tunnel.main.id
  priority            = 1000
}
```

## Deploying

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

OpenTofu manages the complete GCP Cloud VPN configuration including gateways, forwarding rules, tunnels, and routes. The declarative approach ensures consistent VPN connectivity that can be replicated across GCP projects and regions.
