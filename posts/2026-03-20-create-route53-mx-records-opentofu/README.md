# How to Create Route53 MX Records with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Route53, DNS, Email, Infrastructure as Code

Description: Learn how to create Route53 MX records with OpenTofu for email delivery routing to Google Workspace, Microsoft 365, and custom mail servers.

MX records specify the mail servers responsible for accepting email for your domain. Managing them in OpenTofu ensures your email routing is documented, version-controlled, and consistently applied across all domains you manage.

## Hosted Zone Reference

```hcl
data "aws_route53_zone" "main" {
  name         = "example.com."
  private_zone = false
}
```

## Google Workspace MX Records

```hcl
resource "aws_route53_record" "google_mx" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "MX"
  ttl     = 3600

  # Priority (lower = higher priority), mail server hostname
  records = [
    "1 aspmx.l.google.com",
    "5 alt1.aspmx.l.google.com",
    "5 alt2.aspmx.l.google.com",
    "10 alt3.aspmx.l.google.com",
    "10 alt4.aspmx.l.google.com",
  ]
}
```

## Microsoft 365 MX Records

```hcl
resource "aws_route53_record" "m365_mx" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "MX"
  ttl     = 3600

  records = [
    "0 example-com.mail.protection.outlook.com",
  ]
}
```

## Custom Mail Server MX Records

```hcl
resource "aws_route53_record" "custom_mx" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "MX"
  ttl     = 300

  records = [
    "10 mail1.example.com",   # Primary mail server
    "20 mail2.example.com",   # Secondary (backup)
  ]
}

# A records for the mail servers

resource "aws_route53_record" "mail1" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "mail1.example.com"
  type    = "A"
  ttl     = 300
  records = [var.mail1_ip]
}

resource "aws_route53_record" "mail2" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "mail2.example.com"
  type    = "A"
  ttl     = 300
  records = [var.mail2_ip]
}
```

## Subdomain MX Records

```hcl
# Separate mail routing for subdomains
resource "aws_route53_record" "marketing_mx" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "marketing.example.com"
  type    = "MX"
  ttl     = 3600

  records = [
    "10 feedback-smtp.us-east-1.amazonses.com",  # Amazon SES
  ]
}
```

## MX Records for Multiple Domains

```hcl
locals {
  domains = {
    "example.com"    = data.aws_route53_zone.main.zone_id
    "example.io"     = data.aws_route53_zone.io.zone_id
    "example.co.uk"  = data.aws_route53_zone.couk.zone_id
  }

  google_mx_records = [
    "1 aspmx.l.google.com",
    "5 alt1.aspmx.l.google.com",
    "5 alt2.aspmx.l.google.com",
    "10 alt3.aspmx.l.google.com",
    "10 alt4.aspmx.l.google.com",
  ]
}

resource "aws_route53_record" "mx_records" {
  for_each = local.domains

  zone_id = each.value
  name    = each.key
  type    = "MX"
  ttl     = 3600
  records = local.google_mx_records
}
```

## Email Security Records

```hcl
# SPF record (TXT)
resource "aws_route53_record" "spf" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "TXT"
  ttl     = 3600
  records = ["v=spf1 include:_spf.google.com ~all"]
}

# DMARC record
resource "aws_route53_record" "dmarc" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "_dmarc.example.com"
  type    = "TXT"
  ttl     = 3600
  records = ["v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com; pct=100"]
}
```

## Conclusion

Route53 MX records in OpenTofu keep email routing documented alongside your DNS infrastructure. Manage MX records for all your domains with for_each, add SPF and DMARC records for email security, and use TTLs appropriate for how often the records change. Lower TTLs (300s) allow faster updates; higher TTLs (3600s) reduce DNS query load.
