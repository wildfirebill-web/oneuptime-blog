# How to Configure Fastly CDN for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fastly, IPv6, CDN, Edge Computing, VCL, Content Delivery

Description: A guide to configuring Fastly CDN for IPv6 delivery, including service configuration, origin IPv6 connectivity, and VCL for IPv6-aware caching and routing.

Fastly's edge network supports IPv6 natively, providing dual-stack delivery from its global POPs. IPv6 configuration involves setting up backends with IPv6 support and using VCL (Varnish Configuration Language) for IPv6-aware logic.

## Fastly IPv6 Architecture

Fastly's POPs are dual-stack. When a client connects via IPv6:
- Fastly edge server accepts the IPv6 connection
- Fastly connects to your origin using IPv4 or IPv6 (based on backend config)
- Client IPv6 is available in VCL as `client.ip`

## Enabling IPv6 via Fastly Dashboard

1. Log in to https://manage.fastly.com
2. Select your service
3. Go to **Origins** (Backends)
4. For each backend, ensure "Enable IPv6" is checked if your origin has IPv6
5. The service automatically handles IPv6 client connections

## Fastly API Configuration

```bash
# Create a Fastly service with IPv6 backend
FASTLY_API_KEY="your-api-key"
SERVICE_ID="your-service-id"
VERSION="1"

# Add backend with IPv6 support
curl -X POST \
  "https://api.fastly.com/service/${SERVICE_ID}/version/${VERSION}/backend" \
  -H "Fastly-Key: ${FASTLY_API_KEY}" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "name=origin&address=origin.example.com&port=443&use_ssl=1&ssl_check_cert=1"

# Fastly automatically connects to IPv6 if origin has AAAA records
```

## Terraform: Fastly Service with IPv6

```hcl
terraform {
  required_providers {
    fastly = {
      source  = "fastly/fastly"
    }
  }
}

resource "fastly_service_vcl" "main" {
  name = "example-service"

  domain {
    name    = "cdn.example.com"
    comment = "CDN Domain"
  }

  backend {
    address = "origin.example.com"
    name    = "main-origin"
    port    = 443
    use_ssl = true

    # Fastly will use IPv6 if origin has AAAA records
    # No explicit IPv6 flag needed — handled automatically
  }

  # VCL for IPv6-aware logic
  vcl {
    name    = "main"
    content = file("${path.module}/main.vcl")
    main    = true
  }

  force_destroy = true
}
```

## VCL for IPv6-Aware Logic

Fastly's VCL provides `client.ip` which contains the IPv6 address for IPv6 clients:

```vcl
# /etc/fastly/main.vcl

sub vcl_recv {
  # Check if client is connecting via IPv6
  if (client.ip ~ ":") {
    # Client is using IPv6 (contains colons)
    set req.http.X-Client-IP-Version = "IPv6";
    set req.http.X-Client-IPv6 = client.ip;
  } else {
    set req.http.X-Client-IP-Version = "IPv4";
  }

  # Pass to origin with client IP info
  set req.http.X-Forwarded-For = client.ip;
  set req.http.X-Real-IP = client.ip;
}

sub vcl_hash {
  # Optionally include IPv6 in cache key for per-IP-version caching
  # hash_data(regsub(client.ip, ":[^:]+$", ""));  # Hash by /64 prefix
}

sub vcl_deliver {
  # Add diagnostic header showing IP version
  set resp.http.X-Client-Version = req.http.X-Client-IP-Version;
}
```

## Testing Fastly IPv6 Delivery

```bash
# Verify Fastly service has AAAA records
dig AAAA cdn.example.com
# Should return Fastly's IPv6 addresses (e.g., 2a04:4e40::/32)

# Test IPv6 client connection
curl -6 -v https://cdn.example.com/

# Check Fastly response headers
curl -6 -D - https://cdn.example.com/ -o /dev/null | \
  grep -E "X-Served-By|X-Cache|X-Client-Version"

# Verify cache hit vs miss for IPv6
curl -6 https://cdn.example.com/
# First request: X-Cache: MISS
# Second request: X-Cache: HIT
```

## IPv6 Rate Limiting in VCL

```vcl
sub vcl_recv {
  # Rate limit based on /64 prefix for IPv6 (groups addresses by subnet)
  if (client.ip ~ ":") {
    # Extract /64 prefix for rate limiting
    set req.http.X-Client-Prefix = regsub(client.ip, ":[^:]+:[^:]+:[^:]+:[^:]+$", "::/64");
    call check_ipv6_rate_limit;
  }
}
```

Fastly's automatic IPv6 support means minimal configuration is needed — dual-stack delivery is handled by the edge network, and VCL's `client.ip` provides the real IPv6 client address for any custom logic.
