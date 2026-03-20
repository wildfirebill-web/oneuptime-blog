# How to Create DigitalOcean DNS Records with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, DigitalOcean, DNS, Infrastructure as Code, Networking

Description: Learn how to manage DigitalOcean DNS domains and records with OpenTofu, including A, CNAME, MX, and TXT records.

DigitalOcean provides authoritative DNS hosting for domains. OpenTofu lets you manage domains and all record types as code, keeping DNS configuration in version control alongside your infrastructure.

## Adding a Domain

```hcl
resource "digitalocean_domain" "main" {
  name       = "example.com"
  ip_address = digitalocean_droplet.web.ipv4_address  # Creates an A record for the apex
}
```

## Creating DNS Records

### A Record

```hcl
resource "digitalocean_record" "www" {
  domain = digitalocean_domain.main.id
  type   = "A"
  name   = "www"
  value  = digitalocean_droplet.web.ipv4_address
  ttl    = 300
}
```

### CNAME Record

```hcl
resource "digitalocean_record" "api" {
  domain = digitalocean_domain.main.id
  type   = "CNAME"
  name   = "api"
  value  = "myapp.example.com."  # CNAME values must end with a dot
  ttl    = 3600
}
```

### MX Records

```hcl
resource "digitalocean_record" "mx_primary" {
  domain   = digitalocean_domain.main.id
  type     = "MX"
  name     = "@"
  value    = "aspmx.l.google.com."
  priority = 1
  ttl      = 3600
}

resource "digitalocean_record" "mx_secondary" {
  domain   = digitalocean_domain.main.id
  type     = "MX"
  name     = "@"
  value    = "alt1.aspmx.l.google.com."
  priority = 5
  ttl      = 3600
}
```

### TXT Records

```hcl
# SPF record

resource "digitalocean_record" "spf" {
  domain = digitalocean_domain.main.id
  type   = "TXT"
  name   = "@"
  value  = "v=spf1 include:_spf.google.com ~all"
  ttl    = 3600
}

# Domain verification record
resource "digitalocean_record" "verification" {
  domain = digitalocean_domain.main.id
  type   = "TXT"
  name   = "_acme-challenge"
  value  = var.acme_challenge_token
  ttl    = 60
}
```

### CAA Record

```hcl
resource "digitalocean_record" "caa" {
  domain = digitalocean_domain.main.id
  type   = "CAA"
  name   = "@"
  value  = "letsencrypt.org."
  flags  = 0
  tag    = "issue"
  ttl    = 3600
}
```

## Creating Multiple Subdomains with for_each

```hcl
variable "subdomains" {
  type = map(string)
  default = {
    "app"     = "10.0.0.1"
    "api"     = "10.0.0.2"
    "staging" = "10.0.0.3"
  }
}

resource "digitalocean_record" "subdomains" {
  for_each = var.subdomains

  domain = digitalocean_domain.main.id
  type   = "A"
  name   = each.key
  value  = each.value
  ttl    = 300
}
```

## Pointing to a Load Balancer

```hcl
resource "digitalocean_record" "lb" {
  domain = digitalocean_domain.main.id
  type   = "A"
  name   = "@"
  value  = digitalocean_loadbalancer.web.ip
  ttl    = 300
}
```

## Conclusion

DigitalOcean DNS in OpenTofu is managed through `digitalocean_domain` and `digitalocean_record` resources. All common record types are supported - A, AAAA, CNAME, MX, TXT, SRV, CAA, and NS. Use `for_each` for bulk record creation and wire records directly to Droplet IPs or Load Balancer IPs for dynamic DNS management.
