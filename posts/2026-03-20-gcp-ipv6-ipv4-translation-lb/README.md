# How to Configure GCP IPv6-to-IPv4 Translation at Load Balancer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, IPv4, Translation, Load Balancer, Cloud

Description: A guide to GCP's built-in IPv6-to-IPv4 translation at the load balancer layer, enabling IPv6 clients to reach IPv4-only backends without modifying the backend infrastructure.

GCP's Global External HTTP(S) Load Balancer automatically translates IPv6 client connections to IPv4 when forwarding to IPv4-only backends. This is GCP's approach to dual-stack without requiring backends to support IPv6 natively.

## How GCP IPv6-to-IPv4 Translation Works

```
IPv6 Client → GCP Global LB (IPv6 frontend) → IPv4 backend
              ↑ Translation happens here
              - Assigns IPv6 global address
              - Accepts IPv6 connections
              - Translates to IPv4 for backend communication
              - Preserves original IPv6 in X-Forwarded-For
```

## Architecture

The key insight: your backends can remain IPv4-only while clients can connect using IPv6. GCP handles the translation transparently.

## Terraform: IPv6 Frontend with IPv4 Backends

```hcl
# Backend with IPv4 instances (no IPv6 required on backends)
resource "google_compute_backend_service" "main" {
  name     = "main-backend"
  protocol = "HTTP"

  backend {
    group = google_compute_instance_group.ipv4_instances.id
  }

  health_checks = [google_compute_health_check.main.id]

  # No IPv6 configuration required here — backends are IPv4
}

resource "google_compute_health_check" "main" {
  name = "http-health-check"
  http_health_check {
    port         = 80
    request_path = "/health"
  }
}

resource "google_compute_url_map" "main" {
  name            = "main-url-map"
  default_service = google_compute_backend_service.main.id
}

resource "google_compute_target_https_proxy" "main" {
  name             = "main-https-proxy"
  url_map          = google_compute_url_map.main.id
  ssl_certificates = [google_compute_managed_ssl_certificate.main.id]
}

# IPv4 frontend address
resource "google_compute_global_address" "ipv4" {
  name       = "lb-ipv4"
  ip_version = "IPV4"
}

# IPv6 frontend address (for IPv6 clients)
resource "google_compute_global_address" "ipv6" {
  name       = "lb-ipv6"
  ip_version = "IPV6"
}

# IPv4 forwarding rule
resource "google_compute_global_forwarding_rule" "https_ipv4" {
  name       = "https-fwd-ipv4"
  target     = google_compute_target_https_proxy.main.id
  port_range = "443"
  ip_address = google_compute_global_address.ipv4.id
}

# IPv6 forwarding rule → same HTTPS proxy → same IPv4 backends
resource "google_compute_global_forwarding_rule" "https_ipv6" {
  name       = "https-fwd-ipv6"
  target     = google_compute_target_https_proxy.main.id
  port_range = "443"
  ip_address = google_compute_global_address.ipv6.id
}
```

## Reading the Original IPv6 Client IP in Backends

Since backends only see IPv4 (GCP's load balancer IPs), the original IPv6 client address is in HTTP headers:

```bash
# Configure your application to read X-Forwarded-For
# The first IP in X-Forwarded-For is the original client IPv6 address

# Example X-Forwarded-For header when client connects via IPv6:
# X-Forwarded-For: 2001:db8::client, 130.211.x.x

# In nginx:
set $real_client $http_x_forwarded_for;
# Or use the first IP only:
# real_ip_header X-Forwarded-For;
```

## Verifying IPv6 Translation

```bash
# Get your load balancer's IPv6 address
gcloud compute addresses describe lb-ipv6 --global \
  --format="value(address)"

# Test IPv6 client connection
curl -6 https://[GCP_IPV6_ADDRESS]/

# In backend access logs, verify original IPv6 appears in X-Forwarded-For
tail -f /var/log/nginx/access.log | grep -E "2001:|fe80:|::1"
```

## GCP Header Reference

| Header | Contains |
|---|---|
| `X-Forwarded-For` | Original client IP (IPv4 or IPv6) |
| `X-Forwarded-Proto` | Original protocol (http/https) |
| `X-Goog-Iap-Jwt-Assertion` | IAP token if using Cloud IAP |

## Benefits of GCP's Translation Approach

1. **Zero backend changes**: IPv4-only applications work without modification
2. **Gradual migration**: Add IPv6 frontend without touching backends
3. **Same SSL certificate**: One cert works for both IPv4 and IPv6 endpoints
4. **Global anycast**: IPv6 clients reach the nearest GCP point of presence

## Limitations

- Backends cannot initiate connections to IPv6 clients
- Some protocols that embed IP addresses (like FTP active mode) won't work
- True end-to-end IPv6 requires IPv6-capable backend instances

GCP's IPv6-to-IPv4 translation at the load balancer is the recommended starting point for adding IPv6 support to existing GCP workloads — it enables IPv6 connectivity immediately without any infrastructure changes.
