# How to Create AWS WAFv2 IP Sets with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, WAF, IP Sets, Infrastructure as Code

Description: Learn how to create AWS WAFv2 IP sets with OpenTofu to allow or block traffic from specific IP address ranges in your Web ACL rules.

WAFv2 IP sets store collections of IP addresses and CIDR ranges for use in Web ACL rules. Managing them in OpenTofu allows you to version-control your allowlists and blocklists alongside your security rules.

## Creating an IP Set

```hcl
# Allowlist — trusted office and VPN IPs
resource "aws_wafv2_ip_set" "office_ips" {
  name               = "office-and-vpn-ips"
  description        = "Trusted office and VPN IP ranges"
  scope              = "REGIONAL"
  ip_address_version = "IPV4"

  addresses = [
    "203.0.113.0/24",   # US office
    "198.51.100.10/32", # VPN exit node 1
    "198.51.100.11/32", # VPN exit node 2
  ]

  tags = {
    Purpose = "allowlist"
    Team    = "security"
  }
}

# IPv6 IP set
resource "aws_wafv2_ip_set" "office_ips_v6" {
  name               = "office-ips-ipv6"
  description        = "Trusted IPv6 ranges"
  scope              = "REGIONAL"
  ip_address_version = "IPV6"

  addresses = [
    "2001:db8::/32",
  ]
}
```

## Blocklist IP Set

```hcl
resource "aws_wafv2_ip_set" "blocked_ips" {
  name               = "blocked-ips"
  description        = "Known malicious IP addresses"
  scope              = "REGIONAL"
  ip_address_version = "IPV4"

  addresses = var.blocked_ip_ranges  # Managed externally, passed as variable
}
```

## Using IP Sets in Web ACL Rules

```hcl
resource "aws_wafv2_web_acl" "main" {
  name  = "myapp-web-acl"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  # Allow office IPs to access admin endpoints
  rule {
    name     = "AllowAdminFromOffice"
    priority = 1

    action {
      allow {}
    }

    statement {
      and_statement {
        statement {
          ip_set_reference_statement {
            arn = aws_wafv2_ip_set.office_ips.arn
          }
        }

        statement {
          byte_match_statement {
            field_to_match {
              uri_path {}
            }
            positional_constraint = "STARTS_WITH"
            search_string         = "/admin"
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AllowAdminFromOffice"
      sampled_requests_enabled   = true
    }
  }

  # Block admin from non-office IPs
  rule {
    name     = "BlockAdminFromNonOffice"
    priority = 2

    action {
      block {}
    }

    statement {
      and_statement {
        statement {
          not_statement {
            statement {
              ip_set_reference_statement {
                arn = aws_wafv2_ip_set.office_ips.arn
              }
            }
          }
        }

        statement {
          byte_match_statement {
            field_to_match {
              uri_path {}
            }
            positional_constraint = "STARTS_WITH"
            search_string         = "/admin"
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "BlockAdminNonOffice"
      sampled_requests_enabled   = true
    }
  }

  # Block known bad IPs
  rule {
    name     = "BlockKnownBadIPs"
    priority = 5

    action {
      block {}
    }

    statement {
      ip_set_reference_statement {
        arn = aws_wafv2_ip_set.blocked_ips.arn
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "BlockedIPs"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "myapp-web-acl"
    sampled_requests_enabled   = true
  }
}
```

## IP Set from Variable (Dynamic Updates)

```hcl
variable "ci_runner_ips" {
  description = "IP addresses of CI/CD runners that need to bypass rate limits"
  type        = list(string)
}

resource "aws_wafv2_ip_set" "ci_runners" {
  name               = "ci-runner-ips"
  description        = "CI/CD runner IPs for bypass rules"
  scope              = "REGIONAL"
  ip_address_version = "IPV4"
  addresses          = var.ci_runner_ips
}
```

## Conclusion

WAFv2 IP sets in OpenTofu provide version-controlled allowlists and blocklists for your Web ACLs. Store trusted IP ranges for admin access bypass, maintain blocklists of known malicious actors, and use AND statements to combine IP conditions with path conditions for fine-grained rules. Update IP sets independently from Web ACL rules to avoid rule changes when only IPs change.
