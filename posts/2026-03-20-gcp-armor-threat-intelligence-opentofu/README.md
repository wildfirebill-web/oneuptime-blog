# How to Configure GCP Armor Threat Intelligence with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, Cloud Armor, Threat Intelligence, OpenTofu, WAF, Security

Description: Learn how to configure GCP Cloud Armor Threat Intelligence with OpenTofu to leverage Google's threat intelligence feeds for blocking known malicious IPs, Tor exit nodes, and anonymizing proxies.

## Overview

GCP Cloud Armor Threat Intelligence (available with Cloud Armor Managed Protection) automatically blocks traffic from known malicious sources including Tor exit nodes, anonymous proxies, and IP addresses on Google's threat intelligence lists.

## Step 1: Create a Security Policy with Threat Intelligence

```hcl
# main.tf - Cloud Armor security policy with Threat Intelligence

resource "google_compute_security_policy" "threat_intel_policy" {
  name        = "threat-intelligence-policy"
  description = "WAF policy with Threat Intelligence feeds enabled"
  type        = "CLOUD_ARMOR"

  # Adaptive Protection for DDoS and application attacks
  adaptive_protection_config {
    layer_7_ddos_defense_config {
      enable          = true
      rule_visibility = "STANDARD"
    }
  }

  # Rule 1: Block Tor exit nodes
  rule {
    action   = "deny(403)"
    priority = 1000
    description = "Block traffic from Tor exit nodes"

    match {
      expr {
        # Threat intelligence expression for Tor exit nodes
        expression = "evaluateThreatIntelligence('iplist-known-malicious-ips')"
      }
    }
  }

  # Rule 2: Block anonymous proxies and VPNs
  rule {
    action   = "deny(403)"
    priority = 1100
    description = "Block anonymous proxies and VPNs"

    match {
      expr {
        expression = "evaluateThreatIntelligence('iplist-anon-proxies')"
      }
    }
  }

  # Rule 3: Block Tor exit nodes
  rule {
    action   = "deny(403)"
    priority = 1200
    description = "Block Tor exit nodes"

    match {
      expr {
        expression = "evaluateThreatIntelligence('iplist-tor-exit-nodes')"
      }
    }
  }

  # Default allow rule
  rule {
    action   = "allow"
    priority = 2147483647
    description = "Default allow rule"

    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
  }
}
```

## Step 2: Add WAF Preconfigured Rules

```hcl
# Add OWASP Top 10 protection alongside Threat Intelligence
resource "google_compute_security_policy" "combined_policy" {
  name = "combined-security-policy"

  # OWASP SQL Injection protection
  rule {
    action   = "deny(403)"
    priority = 500

    match {
      expr {
        expression = "evaluatePreconfiguredExpr('sqli-stable')"
      }
    }
  }

  # OWASP XSS protection
  rule {
    action   = "deny(403)"
    priority = 510

    match {
      expr {
        expression = "evaluatePreconfiguredExpr('xss-stable')"
      }
    }
  }

  # Threat Intelligence block
  rule {
    action   = "deny(403)"
    priority = 1000

    match {
      expr {
        expression = "evaluateThreatIntelligence('iplist-known-malicious-ips')"
      }
    }
  }

  rule {
    action      = "allow"
    priority    = 2147483647
    description = "Default allow"

    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
  }
}
```

## Step 3: Attach to Load Balancer Backend

```hcl
# Attach the security policy to a backend service
resource "google_compute_backend_service" "protected_backend" {
  name                  = "threat-intel-protected-backend"
  protocol              = "HTTP"
  load_balancing_scheme = "EXTERNAL_MANAGED"

  backend {
    group = google_compute_instance_group_manager.web_mig.instance_group
  }

  health_checks = [google_compute_health_check.http_hc.id]

  # Apply the Cloud Armor security policy
  security_policy = google_compute_security_policy.combined_policy.id
}
```

## Summary

GCP Cloud Armor Threat Intelligence with OpenTofu integrates Google's threat intelligence feeds directly into your WAF policy. By blocking Tor exit nodes, anonymous proxies, and known malicious IPs alongside OWASP protections, you create a comprehensive defense-in-depth approach that reduces attack surface without manually maintaining IP blocklists.
