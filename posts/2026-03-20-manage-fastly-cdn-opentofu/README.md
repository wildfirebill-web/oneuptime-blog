# How to Manage Fastly CDN Resources with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Fastly, CDN, Edge, Caching

Description: Learn how to manage Fastly CDN services, backends, VCL configurations, and TLS settings using OpenTofu for reproducible content delivery network infrastructure.

## Introduction

The Fastly provider for OpenTofu manages Fastly services including backends, domains, request/response conditions, cache settings, headers, and VCL snippets. Managing CDN configuration as code prevents the configuration drift that commonly occurs when CDN settings are modified directly through the Fastly dashboard.

## Provider Configuration

```hcl
terraform {
  required_providers {
    fastly = {
      source  = "fastly/fastly"
      version = "~> 5.0"
    }
  }
}

provider "fastly" {
  api_key = var.fastly_api_key
}
```

## Service with Backends

```hcl
resource "fastly_service_vcl" "app" {
  name          = "app-cdn"
  force_destroy = false

  domain {
    name    = "www.example.com"
    comment = "Main application domain"
  }

  domain {
    name    = "cdn.example.com"
    comment = "CDN subdomain"
  }

  backend {
    address           = "app.origin.example.com"
    name              = "app-origin"
    port              = 443
    use_ssl           = true
    ssl_cert_hostname = "app.origin.example.com"
    ssl_sni_hostname  = "app.origin.example.com"
    connect_timeout   = 5000
    between_bytes_timeout = 10000
    first_byte_timeout    = 15000
    max_connections       = 200
    weight                = 100
  }

  backend {
    address = "app-failover.origin.example.com"
    name    = "app-failover"
    port    = 443
    use_ssl = true
    weight  = 1  # Low weight = failover
  }

  # Health check
  healthcheck {
    name       = "app-health"
    host       = "app.origin.example.com"
    path       = "/health"
    method     = "GET"
    expected_response = 200
    check_interval = 5000
    timeout        = 2000
    threshold      = 3
    initial        = 3
    window         = 5
  }

  force_destroy = false
}
```

## Cache Settings and Headers

```hcl
resource "fastly_service_vcl" "app" {
  # ... other config ...

  # Cache TTL settings
  cache_setting {
    name          = "api-no-cache"
    action        = "pass"
    stale_ttl     = 0
    ttl           = 0
  }

  # Custom response headers
  header {
    name              = "Remove Server Header"
    action            = "delete"
    type              = "response"
    dst               = "http.Server"
    ignore_if_set     = false
    priority          = 100
  }

  header {
    name          = "Set HSTS"
    action        = "set"
    type          = "response"
    dst           = "http.Strict-Transport-Security"
    source        = "\"max-age=31536000; includeSubDomains\""
    priority      = 100
  }

  # Request condition to identify API traffic
  condition {
    name      = "is-api-request"
    type      = "REQUEST"
    statement = "req.url ~ \"^/api/\""
    priority  = 10
  }

  # Apply no-cache to API requests
  cache_setting {
    name          = "api-cache-bypass"
    action        = "pass"
    cache_condition = "is-api-request"
  }
}
```

## VCL Snippets for Custom Logic

```hcl
resource "fastly_service_vcl" "app" {
  # ... other config ...

  snippet {
    name     = "redirect-http-to-https"
    type     = "recv"
    priority = 100
    content  = <<VCL
if (req.http.X-Forwarded-Proto != "https") {
  error 801 "Force SSL";
}
VCL
  }

  snippet {
    name     = "deliver-redirect"
    type     = "error"
    priority = 100
    content  = <<VCL
if (obj.status == 801) {
  set obj.status = 301;
  set obj.response = "Moved Permanently";
  set obj.http.Location = "https://" + req.http.host + req.url;
  deliver;
}
VCL
  }
}
```

## TLS Configuration

```hcl
resource "fastly_tls_subscription" "app" {
  domains               = ["www.example.com", "cdn.example.com"]
  certificate_authority = "lets-encrypt"
}

resource "fastly_tls_subscription_validation" "app" {
  subscription_id = fastly_tls_subscription.app.id
}
```

## Activating the Service Version

```hcl
resource "fastly_service_vcl" "app" {
  # ... other config ...

  activate = true  # Activate the latest version automatically
}
```

## Conclusion

Fastly CDN configuration managed with OpenTofu ensures that CDN behavior - caching rules, security headers, backend routing, and VCL snippets - goes through the same review process as application code. The `activate = true` attribute ensures new versions are immediately published after apply, while the Fastly version history provides rollback capability if a configuration change causes issues.
