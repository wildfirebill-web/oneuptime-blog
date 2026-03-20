# How to Configure GCP Cloud Load Balancing with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, Load Balancing, Google Cloud, Dual-Stack, HTTP Load Balancer

Description: Configure Google Cloud HTTP(S), TCP, and Network load balancers to accept IPv6 client connections and route traffic to IPv4 backends using GCP's built-in IPv6 termination.

## Introduction

Google Cloud Load Balancing supports IPv6 at the frontend through Global HTTP(S) Load Balancers and Network Load Balancers. The global HTTP(S) load balancer accepts IPv6 connections from clients and translates them to IPv4 when forwarding to backends - a process called IPv6 termination. This allows backends to remain IPv4-only while clients connect over IPv6.

## Global HTTP(S) Load Balancer with IPv6

```bash
PROJECT="my-project"

# Step 1: Reserve a global IPv6 address

gcloud compute addresses create lb-ipv6-vip \
    --project="$PROJECT" \
    --network-tier=PREMIUM \
    --ip-version=IPV6 \
    --global

# Get the assigned IPv6 address
gcloud compute addresses describe lb-ipv6-vip \
    --project="$PROJECT" \
    --global \
    --format="get(address)"

# Step 2: Create backend service
gcloud compute backend-services create web-backend \
    --project="$PROJECT" \
    --protocol=HTTP \
    --port-name=http \
    --health-checks=http-health-check \
    --global

# Step 3: Create URL map
gcloud compute url-maps create web-url-map \
    --project="$PROJECT" \
    --default-service=web-backend

# Step 4: Create HTTPS proxy with SSL certificate
gcloud compute target-https-proxies create web-https-proxy \
    --project="$PROJECT" \
    --url-map=web-url-map \
    --ssl-certificates=my-ssl-cert

# Step 5: Create forwarding rule for IPv6
gcloud compute forwarding-rules create web-ipv6-rule \
    --project="$PROJECT" \
    --address=lb-ipv6-vip \
    --target-https-proxy=web-https-proxy \
    --ports=443 \
    --global
```

## Terraform Global HTTP(S) LB with IPv6

```hcl
# lb_ipv6.tf

# Reserve global IPv6 address
resource "google_compute_global_address" "ipv6" {
  name         = "lb-ipv6-vip"
  project      = var.project_id
  ip_version   = "IPV6"
  network_tier = "PREMIUM"
  address_type = "EXTERNAL"
}

# Instance group as backend
resource "google_compute_instance_group" "web" {
  name    = "web-group"
  zone    = "us-east1-b"
  project = var.project_id

  instances = [google_compute_instance.web.self_link]

  named_port {
    name = "http"
    port = 80
  }
}

# Backend service
resource "google_compute_backend_service" "web" {
  name                  = "web-backend"
  project               = var.project_id
  protocol              = "HTTP"
  port_name             = "http"
  load_balancing_scheme = "EXTERNAL"
  health_checks         = [google_compute_health_check.http.id]

  backend {
    group = google_compute_instance_group.web.id
  }
}

# URL map
resource "google_compute_url_map" "web" {
  name            = "web-url-map"
  project         = var.project_id
  default_service = google_compute_backend_service.web.id
}

# HTTPS proxy
resource "google_compute_target_https_proxy" "web" {
  name             = "web-https-proxy"
  project          = var.project_id
  url_map          = google_compute_url_map.web.id
  ssl_certificates = [google_compute_ssl_certificate.web.id]
}

# IPv6 forwarding rule
resource "google_compute_global_forwarding_rule" "ipv6" {
  name       = "web-ipv6-rule"
  project    = var.project_id
  target     = google_compute_target_https_proxy.web.id
  port_range = "443"
  ip_address = google_compute_global_address.ipv6.address
  ip_version = "IPV6"
}

# IPv4 forwarding rule (keep both)
resource "google_compute_global_address" "ipv4" {
  name         = "lb-ipv4-vip"
  project      = var.project_id
  ip_version   = "IPV4"
  network_tier = "PREMIUM"
  address_type = "EXTERNAL"
}

resource "google_compute_global_forwarding_rule" "ipv4" {
  name       = "web-ipv4-rule"
  project    = var.project_id
  target     = google_compute_target_https_proxy.web.id
  port_range = "443"
  ip_address = google_compute_global_address.ipv4.address
}
```

## Network Load Balancer with IPv6

```bash
# Regional NLB with IPv6 frontend
# Step 1: Reserve regional IPv6 address
gcloud compute addresses create nlb-ipv6 \
    --project="$PROJECT" \
    --region=us-east1 \
    --ip-version=IPV6 \
    --network-tier=PREMIUM

# Step 2: Create forwarding rule for TCP NLB
gcloud compute forwarding-rules create nlb-ipv6-rule \
    --project="$PROJECT" \
    --region=us-east1 \
    --address=nlb-ipv6 \
    --target-pool=web-pool \
    --ports=80,443 \
    --ip-protocol=TCP

# Check the IPv6 forwarding rule
gcloud compute forwarding-rules describe nlb-ipv6-rule \
    --project="$PROJECT" \
    --region=us-east1
```

## Verify Load Balancer IPv6 Connectivity

```bash
# Get IPv6 VIP address
IPV6_VIP=$(gcloud compute addresses describe lb-ipv6-vip \
    --project="$PROJECT" \
    --global \
    --format="get(address)")

echo "LB IPv6 VIP: $IPV6_VIP"

# Test HTTP over IPv6
curl -6 -v "https://[$IPV6_VIP]/" \
    --resolve "example.com:443:[$IPV6_VIP]" \
    -H "Host: example.com"

# Check AAAA DNS record points to LB IPv6
dig AAAA example.com

# Test connectivity from IPv6-only client
curl -6 https://example.com/
```

## Conclusion

GCP Global HTTP(S) Load Balancers accept IPv6 connections natively by reserving a global IPv6 address with `ip_version = "IPV6"` and creating an IPv6 forwarding rule pointing to your HTTPS proxy. Backends remain IPv4, as GCP handles IPv6 termination at the frontend. Use both IPv4 and IPv6 forwarding rules pointing to the same proxy for dual-stack support. Add AAAA DNS records pointing to the IPv6 VIP for full IPv6 client accessibility.
