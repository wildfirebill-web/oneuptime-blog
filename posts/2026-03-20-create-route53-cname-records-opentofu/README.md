# How to Create Route53 CNAME Records with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Route53, DNS, CNAME, Infrastructure as Code

Description: Learn how to create Route53 CNAME records with OpenTofu for service discovery, SSL validation, and third-party service integration.

CNAME records map one domain name to another. They're used for subdomains pointing to external services, ACM certificate validation, and service aliases. Managing them in OpenTofu keeps DNS configuration version-controlled alongside the services they point to.

## Hosted Zone Reference

```hcl
data "aws_route53_zone" "main" {
  name         = "example.com."
  private_zone = false
}
```

## Basic CNAME Record

```hcl
resource "aws_route53_record" "www" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "CNAME"
  ttl     = 300

  records = ["example.com"]
}
```

## CNAME for Third-Party Services

```hcl
# Zendesk custom domain
resource "aws_route53_record" "support" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "support.example.com"
  type    = "CNAME"
  ttl     = 300

  records = ["example.zendesk.com"]
}

# Intercom custom domain
resource "aws_route53_record" "help" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "help.example.com"
  type    = "CNAME"
  ttl     = 300

  records = ["custom.intercom.help"]
}
```

## ACM Certificate Validation CNAMEs

```hcl
resource "aws_acm_certificate" "main" {
  domain_name               = "example.com"
  subject_alternative_names = ["*.example.com"]
  validation_method         = "DNS"
}

# Automatically create the validation CNAME records
resource "aws_route53_record" "acm_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  zone_id = data.aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  ttl     = 60

  records = [each.value.record]
}

resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.acm_validation : record.fqdn]
}
```

## CNAME for Internal Services

```hcl
data "aws_route53_zone" "internal" {
  name         = "internal.example.com."
  private_zone = true
}

resource "aws_route53_record" "db_primary" {
  zone_id = data.aws_route53_zone.internal.zone_id
  name    = "db.internal.example.com"
  type    = "CNAME"
  ttl     = 60

  records = [aws_db_instance.primary.address]
}
```

## Multiple CNAMEs with for_each

```hcl
locals {
  cname_records = {
    "mail"      = "mail.google.com"
    "calendar"  = "calendar.google.com"
    "docs"      = "docs.google.com"
    "autodiscover" = "autodiscover.outlook.com"
  }
}

resource "aws_route53_record" "cnames" {
  for_each = local.cname_records

  zone_id = data.aws_route53_zone.main.zone_id
  name    = "${each.key}.example.com"
  type    = "CNAME"
  ttl     = 300

  records = [each.value]
}
```

## Weighted CNAME Records

```hcl
resource "aws_route53_record" "api_blue" {
  zone_id        = data.aws_route53_zone.main.zone_id
  name           = "api.example.com"
  type           = "CNAME"
  ttl            = 60
  set_identifier = "blue"

  weighted_routing_policy {
    weight = 90  # 90% of traffic
  }

  records = ["api-blue.internal.example.com"]
}

resource "aws_route53_record" "api_green" {
  zone_id        = data.aws_route53_zone.main.zone_id
  name           = "api.example.com"
  type           = "CNAME"
  ttl            = 60
  set_identifier = "green"

  weighted_routing_policy {
    weight = 10  # 10% of traffic (canary)
  }

  records = ["api-green.internal.example.com"]
}
```

## Conclusion

Route53 CNAME records in OpenTofu keep your DNS aliases version-controlled and consistently applied. Use for_each to manage multiple CNAMEs, automate ACM certificate validation records, and leverage weighted routing for blue-green deployments. Always remember that CNAMEs cannot be used at the zone apex — use alias records there instead.
