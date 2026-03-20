# How to Create Route53 TXT Records with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Route53, DNS, TXT Records, Infrastructure as Code

Description: Learn how to create Route53 TXT records with OpenTofu for domain verification, SPF, DKIM, DMARC, and other DNS-based authentication.

TXT records carry arbitrary text data used for domain ownership verification, email authentication, and service configuration. Managing them in OpenTofu prevents accidental deletion of critical records and documents their purpose.

## Hosted Zone Reference

```hcl
data "aws_route53_zone" "main" {
  name         = "example.com."
  private_zone = false
}
```

## Domain Ownership Verification

```hcl
# Google Search Console verification
resource "aws_route53_record" "google_verify" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "TXT"
  ttl     = 3600
  records = ["google-site-verification=abcdef1234567890"]
}
```

## SPF Record

```hcl
resource "aws_route53_record" "spf" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "TXT"
  ttl     = 3600

  # Allow Google Workspace and SendGrid to send on behalf of this domain
  records = [
    "v=spf1 include:_spf.google.com include:sendgrid.net ~all"
  ]
}
```

## DKIM Record

```hcl
# Google Workspace DKIM — key obtained from Google Admin Console
resource "aws_route53_record" "dkim_google" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "google._domainkey.example.com"
  type    = "TXT"
  ttl     = 3600

  records = [
    "v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA..."
  ]
}

# SendGrid DKIM
resource "aws_route53_record" "dkim_sendgrid" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "s1._domainkey.example.com"
  type    = "CNAME"  # SendGrid uses CNAME, not TXT
  ttl     = 300
  records = ["s1.domainkey.u12345678.wl001.sendgrid.net"]
}
```

## DMARC Record

```hcl
resource "aws_route53_record" "dmarc" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "_dmarc.example.com"
  type    = "TXT"
  ttl     = 3600

  records = [
    # p=reject: reject emails that fail DMARC
    # rua: aggregate report destination
    # ruf: forensic report destination
    "v=DMARC1; p=reject; rua=mailto:dmarc@example.com; ruf=mailto:dmarc@example.com; fo=1; pct=100"
  ]
}
```

## Multiple TXT Records on the Same Name

```hcl
resource "aws_route53_record" "root_txt" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "TXT"
  ttl     = 3600

  # Multiple values in one record set
  records = [
    "v=spf1 include:_spf.google.com ~all",
    "google-site-verification=abcdef1234567890",
    "MS=ms12345678",  # Microsoft verification
    "atlassian-domain-verification=abcdefgh",
  ]
}
```

## TXT Records from Variables

```hcl
variable "domain_verification_records" {
  description = "Domain verification TXT record values"
  type = map(string)
  default = {
    google    = "google-site-verification=xxx"
    microsoft = "MS=msXXXXXX"
    atlassian = "atlassian-domain-verification=xxx"
  }
}

# If all verifications go on the root domain
resource "aws_route53_record" "verifications" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "TXT"
  ttl     = 3600
  records = values(var.domain_verification_records)
}
```

## BIMI Record (Brand Indicator for Message Identification)

```hcl
resource "aws_route53_record" "bimi" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "default._bimi.example.com"
  type    = "TXT"
  ttl     = 3600

  records = [
    "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/bimi.pem"
  ]
}
```

## Conclusion

Route53 TXT records in OpenTofu ensure your domain verification and email authentication records are version-controlled. Consolidate multiple verifications into a single record set, document the purpose of each value in comments, and use variables for records that change per-environment. Never manually delete TXT records without checking OpenTofu state first.
