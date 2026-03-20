# How to Design a Certificate Module for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, ACM, TLS, AWS, Modules, Certificates

Description: Learn how to design a reusable ACM certificate module for OpenTofu that handles certificate request, DNS validation, and automatic renewal configuration.

## Introduction

TLS certificate management with ACM involves requesting a certificate, creating DNS validation records, waiting for validation, and wiring the certificate to your load balancer or CloudFront distribution. A certificate module encapsulates this entire lifecycle.

## variables.tf

```hcl
variable "domain_name"               { type = string }
variable "subject_alternative_names" { type = list(string); default = [] }
variable "route53_zone_id"           { type = string }
variable "validation_method"         { type = string; default = "DNS" }
variable "environment"               { type = string }

variable "wait_for_validation" {
  description = "Wait for certificate to be issued before module completes"
  type        = bool
  default     = true
}

variable "tags" { type = map(string); default = {} }
```

## main.tf

```hcl
locals {
  tags = merge({ Environment = var.environment, ManagedBy = "OpenTofu" }, var.tags)
}

# Request the ACM certificate

resource "aws_acm_certificate" "main" {
  domain_name               = var.domain_name
  subject_alternative_names = var.subject_alternative_names
  validation_method         = var.validation_method

  tags = merge(local.tags, { Name = var.domain_name })

  lifecycle {
    # Create a new cert before destroying the old one to avoid downtime
    create_before_destroy = true
  }
}

# Create Route53 DNS validation records for each domain
resource "aws_route53_record" "validation" {
  # One record per unique validation option
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  zone_id         = var.route53_zone_id
  name            = each.value.name
  type            = each.value.type
  records         = [each.value.record]
  ttl             = 60
  allow_overwrite = true
}

# Wait for ACM to validate the certificate via DNS
resource "aws_acm_certificate_validation" "main" {
  count           = var.wait_for_validation ? 1 : 0
  certificate_arn = aws_acm_certificate.main.arn
  validation_record_fqdns = [
    for record in aws_route53_record.validation : record.fqdn
  ]
}
```

## outputs.tf

```hcl
output "certificate_arn" {
  # Return validated ARN if waiting for validation, otherwise the raw ARN
  value = var.wait_for_validation ? (
    aws_acm_certificate_validation.main[0].certificate_arn
  ) : (
    aws_acm_certificate.main.arn
  )
}

output "domain_name"         { value = aws_acm_certificate.main.domain_name }
output "status"              { value = aws_acm_certificate.main.status }
output "validation_record_fqdns" {
  value = [for r in aws_route53_record.validation : r.fqdn]
}
```

## Example Usage

```hcl
module "cert" {
  source          = "./modules/certificate"
  domain_name     = "example.com"
  subject_alternative_names = ["www.example.com", "api.example.com"]
  route53_zone_id = module.dns.zone_id
  environment     = var.environment
}

# Use the certificate in a load balancer listener
resource "aws_lb_listener" "https" {
  certificate_arn = module.cert.certificate_arn
  # ...
}
```

## Conclusion

The certificate module handles the full ACM workflow: request, DNS validation record creation, and validation waiting. The `create_before_destroy` lifecycle rule is critical - without it, replacing a certificate would briefly leave your service without TLS termination. The `wait_for_validation` flag lets you skip blocking in CI pipelines where you know validation will complete later.
