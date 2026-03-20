# How to Create DigitalOcean Load Balancers with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, DigitalOcean, Load Balancers, Infrastructure as Code, Networking

Description: Learn how to create DigitalOcean Load Balancers with OpenTofu, including forwarding rules, health checks, and SSL termination.

DigitalOcean Load Balancers distribute traffic across Droplets. OpenTofu lets you define forwarding rules, health checks, and SSL termination as code and wire load balancers to Droplets via tags.

## Creating a Basic HTTP Load Balancer

```hcl
resource "digitalocean_loadbalancer" "web" {
  name   = "web-lb"
  region = "nyc3"

  # Route HTTP traffic to port 80 on backend Droplets
  forwarding_rule {
    entry_port     = 80
    entry_protocol = "http"
    target_port    = 80
    target_protocol = "http"
  }

  # Health check configuration
  healthcheck {
    port     = 80
    protocol = "http"
    path     = "/health"
    check_interval_seconds   = 10
    response_timeout_seconds = 5
    unhealthy_threshold      = 3
    healthy_threshold        = 5
  }

  # Target Droplets tagged "web"
  droplet_tag = "web"
}

output "lb_ip" {
  value = digitalocean_loadbalancer.web.ip
}
```

## Adding HTTPS with SSL Termination

```hcl
# Use a DigitalOcean-managed Let's Encrypt certificate
resource "digitalocean_certificate" "web" {
  name    = "web-cert"
  type    = "lets_encrypt"
  domains = ["example.com", "www.example.com"]
}

resource "digitalocean_loadbalancer" "https" {
  name   = "https-lb"
  region = "nyc3"

  # HTTPS forwarding rule using the certificate
  forwarding_rule {
    entry_port      = 443
    entry_protocol  = "https"
    target_port     = 80
    target_protocol = "http"
    certificate_name = digitalocean_certificate.web.name
  }

  # Redirect HTTP to HTTPS
  redirect_http_to_https = true

  healthcheck {
    port     = 80
    protocol = "http"
    path     = "/health"
  }

  droplet_tag = "web"
}
```

## Sticky Sessions

```hcl
resource "digitalocean_loadbalancer" "sticky" {
  name   = "sticky-lb"
  region = "nyc3"

  forwarding_rule {
    entry_port      = 80
    entry_protocol  = "http"
    target_port     = 8080
    target_protocol = "http"
  }

  # Enable cookie-based sticky sessions
  sticky_sessions {
    type               = "cookies"
    cookie_name        = "lb_session"
    cookie_ttl_seconds = 300
  }

  healthcheck {
    port     = 8080
    protocol = "http"
    path     = "/ping"
  }

  droplet_tag = "app"
}
```

## Placing Load Balancer in a VPC

```hcl
resource "digitalocean_vpc" "main" {
  name     = "production-vpc"
  region   = "nyc3"
  ip_range = "10.10.0.0/16"
}

resource "digitalocean_loadbalancer" "internal" {
  name     = "internal-lb"
  region   = "nyc3"
  vpc_uuid = digitalocean_vpc.main.id

  forwarding_rule {
    entry_port      = 80
    entry_protocol  = "http"
    target_port     = 8080
    target_protocol = "http"
  }

  healthcheck {
    port     = 8080
    protocol = "http"
    path     = "/health"
  }

  droplet_tag = "app"
}
```

## Conclusion

DigitalOcean Load Balancers in OpenTofu are configured through forwarding rules, health checks, and Droplet tag associations. Add SSL certificates for HTTPS termination, enable HTTP-to-HTTPS redirect for secure configurations, and use sticky sessions when your application requires session affinity.
