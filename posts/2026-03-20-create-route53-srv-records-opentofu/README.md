# How to Create Route53 SRV Records with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Route53, DNS, SRV Records, Infrastructure as Code

Description: Learn how to create Route53 SRV records with OpenTofu for service discovery, SIP, XMPP, and other protocol-specific DNS routing.

SRV records specify the location (hostname and port) of services. They're used for service discovery, SIP telephony, XMPP messaging, and Kubernetes-style DNS-based service endpoints. Managing them in OpenTofu keeps service endpoint configuration version-controlled.

## SRV Record Format

SRV records follow the format: `priority weight port target`

```text
_service._proto.name  TTL  IN  SRV  priority  weight  port  target
```

## Basic SRV Record

```hcl
data "aws_route53_zone" "main" {
  name         = "example.com."
  private_zone = false
}

resource "aws_route53_record" "sip_tcp" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "_sip._tcp.example.com"
  type    = "SRV"
  ttl     = 300

  records = [
    "10 60 5060 sip1.example.com",  # priority=10, weight=60, port=5060
    "20 40 5060 sip2.example.com",  # priority=20, weight=40, port=5060
  ]
}
```

## XMPP Service Records

```hcl
# Client-to-server XMPP

resource "aws_route53_record" "xmpp_client" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "_xmpp-client._tcp.example.com"
  type    = "SRV"
  ttl     = 300
  records = ["5 0 5222 xmpp.example.com"]
}

# Server-to-server XMPP federation
resource "aws_route53_record" "xmpp_server" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "_xmpp-server._tcp.example.com"
  type    = "SRV"
  ttl     = 300
  records = ["5 0 5269 xmpp.example.com"]
}
```

## Microsoft Teams / Skype for Business

```hcl
resource "aws_route53_record" "sipfederationtls" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "_sipfederationtls._tcp.example.com"
  type    = "SRV"
  ttl     = 3600
  records = ["100 1 5061 sipfed.online.lync.com"]
}

resource "aws_route53_record" "sip_tls" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "_sip._tls.example.com"
  type    = "SRV"
  ttl     = 3600
  records = ["100 1 443 sipdir.online.lync.com"]
}
```

## Service Discovery for Microservices

```hcl
# Service discovery for internal API services
data "aws_route53_zone" "internal" {
  name         = "internal.example.com."
  private_zone = true
}

resource "aws_route53_record" "api_service" {
  zone_id = data.aws_route53_zone.internal.zone_id
  name    = "_api._tcp.internal.example.com"
  type    = "SRV"
  ttl     = 60  # Low TTL for dynamic service discovery

  records = [
    "10 100 8080 api-1.internal.example.com",
    "10 100 8080 api-2.internal.example.com",
    "10 100 8080 api-3.internal.example.com",
  ]
}
```

## Dynamic SRV Records with for_each

```hcl
locals {
  api_instances = {
    "api-1" = "10.0.1.10"
    "api-2" = "10.0.1.11"
    "api-3" = "10.0.1.12"
  }
}

# A records for each instance
resource "aws_route53_record" "api_instances" {
  for_each = local.api_instances

  zone_id = data.aws_route53_zone.internal.zone_id
  name    = "${each.key}.internal.example.com"
  type    = "A"
  ttl     = 60
  records = [each.value]
}

# SRV record pointing to all instances
resource "aws_route53_record" "api_srv" {
  zone_id = data.aws_route53_zone.internal.zone_id
  name    = "_api._tcp.internal.example.com"
  type    = "SRV"
  ttl     = 60

  records = [
    for name in keys(local.api_instances) :
    "10 100 8080 ${name}.internal.example.com"
  ]
}
```

## LDAP Service Record

```hcl
resource "aws_route53_record" "ldap" {
  zone_id = data.aws_route53_zone.internal.zone_id
  name    = "_ldap._tcp.internal.example.com"
  type    = "SRV"
  ttl     = 300
  records = [
    "0 100 389 ldap.internal.example.com",  # Primary LDAP
  ]
}
```

## Conclusion

Route53 SRV records in OpenTofu enable DNS-based service discovery for protocols that support it. Define SRV records for SIP, XMPP, LDAP, and custom microservices, and use dynamic generation with for_each when the set of service instances is managed as code. Keep TTLs low for services that change frequently to ensure fast propagation.
