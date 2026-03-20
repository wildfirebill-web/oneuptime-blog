# How to Handle Resources That Require Manual Steps in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Manual Steps, null_resource, Provisioners, Infrastructure as Code, Best Practices

Description: Learn how to handle resources that require out-of-band manual steps using null_resource, local-exec provisioners, and preconditions in OpenTofu.

## Introduction

Some resources require human action to complete provisioning — DNS propagation confirmation, SSL certificate validation, third-party vendor activation, or regulatory approvals. OpenTofu can flag these requirements, pause workflows, and verify completion before proceeding.

## Documenting Manual Steps with Preconditions

Use `precondition` blocks to verify that required manual steps have been completed.

```hcl
variable "dns_propagated" {
  type        = bool
  description = "Set to true after DNS records have propagated and been verified"
  default     = false
}

resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]

  timeouts {
    create = "5m"
  }

  lifecycle {
    precondition {
      condition     = var.dns_propagated
      error_message = "Set dns_propagated=true after verifying DNS records have propagated. Check with: dig ${aws_acm_certificate.main.domain_name}"
    }
  }
}
```

## Null Resource for Manual Verification

Pause execution with a `local-exec` prompt.

```hcl
resource "null_resource" "verify_manual_step" {
  depends_on = [aws_acm_certificate.main]

  # Only run when certificate is first created
  triggers = {
    certificate_arn = aws_acm_certificate.main.arn
  }

  provisioner "local-exec" {
    command = <<-SCRIPT
      echo ""
      echo "═══════════════════════════════════════════════════════"
      echo "MANUAL STEP REQUIRED"
      echo "═══════════════════════════════════════════════════════"
      echo ""
      echo "Add the following DNS validation record to your domain:"
      echo ""
      echo "  Name:  ${aws_acm_certificate.main.domain_validation_options[*].resource_record_name}"
      echo "  Type:  CNAME"
      echo "  Value: ${aws_acm_certificate.main.domain_validation_options[*].resource_record_value}"
      echo ""
      echo "Press Enter after the DNS record has been added..."
      read confirmation
    SCRIPT
    interpreter = ["/bin/bash", "-c"]
  }
}
```

## Using Outputs to Guide Manual Steps

```hcl
output "manual_steps_required" {
  description = "Manual actions required before the next apply"
  value = <<-EOT
    1. Add the following CNAME record to your DNS:
       Name:  ${aws_acm_certificate.main.domain_validation_options[0].resource_record_name}
       Value: ${aws_acm_certificate.main.domain_validation_options[0].resource_record_value}

    2. After DNS propagates (5-30 minutes), run:
       tofu apply -var="dns_propagated=true"

    3. Log in to the vendor portal at https://vendor.example.com
       and activate the subscription for account ${data.aws_caller_identity.current.account_id}
  EOT
}
```

## Skip Flag Pattern

Allow skipping verification in automated environments.

```hcl
variable "skip_manual_verification" {
  type        = bool
  description = "Set to true in CI environments where manual steps are pre-completed"
  default     = false
}

resource "null_resource" "manual_verification" {
  count = var.skip_manual_verification ? 0 : 1

  provisioner "local-exec" {
    command = <<-SCRIPT
      echo "Manual verification required. Follow the runbook at:"
      echo "https://wiki.example.com/runbooks/opentofu-deploy"
      echo ""
      echo "Press Enter to continue after completing verification..."
      read confirmation
    SCRIPT
    interpreter = ["/bin/bash", "-c"]
  }
}
```

## Phased Apply with -target

For complex deployments with mandatory review between phases, use `-target` to apply in stages.

```bash
# Phase 1: Create the certificate
tofu apply -target=aws_acm_certificate.main

# Output the DNS record to add
tofu output manual_steps_required

# After manual DNS changes are complete and verified...
# Phase 2: Complete the deployment
tofu apply -var="dns_propagated=true"
```

## Summary

Resources requiring manual steps are handled in OpenTofu through precondition variables that gate progress, null_resource prompts that pause execution, descriptive outputs that guide operators, and phased apply strategies. These patterns make the manual requirements explicit, auditable, and hard to skip accidentally.
