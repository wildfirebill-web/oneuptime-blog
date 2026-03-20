# How to Configure IPv6 Firewall Rules on GCP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, Firewall, VPC, Google Cloud, Security

Description: Create and manage Google Cloud VPC firewall rules for IPv6 traffic, configure ICMPv6 rules, and set up secure dual-stack firewall policies for GCP resources.

## Introduction

Google Cloud VPC firewall rules apply to both IPv4 and IPv6 traffic but require separate rules for each protocol. IPv6 firewall rules use `::/0` as the source or destination range for all IPv6 traffic. Critical ICMPv6 types must be allowed for basic IPv6 operations such as Neighbor Discovery Protocol (NDP) and Path MTU Discovery.

## Basic IPv6 Firewall Rules with gcloud

```bash
PROJECT="my-project"
VPC_NAME="vpc-main"

# Allow all inbound IPv6 traffic (use with caution)
gcloud compute firewall-rules create allow-all-ipv6 \
    --network="$VPC_NAME" \
    --direction=INGRESS \
    --priority=1000 \
    --source-ranges="::/0" \
    --rules=all \
    --project="$PROJECT"

# Allow HTTP and HTTPS over IPv6 to web servers
gcloud compute firewall-rules create allow-web-ipv6 \
    --network="$VPC_NAME" \
    --direction=INGRESS \
    --priority=1000 \
    --source-ranges="::/0" \
    --rules=tcp:80,tcp:443 \
    --target-tags=web-server \
    --project="$PROJECT"

# Allow SSH over IPv6 from specific prefix
gcloud compute firewall-rules create allow-ssh-ipv6 \
    --network="$VPC_NAME" \
    --direction=INGRESS \
    --priority=1000 \
    --source-ranges="2001:db8:admin::/48" \
    --rules=tcp:22 \
    --target-tags=ssh-allowed \
    --project="$PROJECT"
```

## ICMPv6 Rules (Required for IPv6 Operations)

```bash
# Allow essential ICMPv6 types for NDP and PMTUD
# ICMPv6 types needed: 133-137 (NDP), 2 (Packet Too Big), 128 (echo request)
gcloud compute firewall-rules create allow-icmpv6-essential \
    --network="$VPC_NAME" \
    --direction=INGRESS \
    --priority=900 \
    --source-ranges="::/0" \
    --rules=icmpv6 \
    --project="$PROJECT"

# GCP simplification: use protocol name 'icmpv6'
# This allows all ICMPv6 types — you cannot filter by ICMPv6 type in basic firewall rules
# Use Cloud Armor or Google Cloud IDS for granular ICMPv6 filtering

# Verify the rule
gcloud compute firewall-rules describe allow-icmpv6-essential \
    --project="$PROJECT"
```

## Terraform Firewall Rules for IPv6

```hcl
# firewall_ipv6.tf

# Allow web traffic over IPv6
resource "google_compute_firewall" "allow_web_ipv6" {
  name    = "allow-web-ipv6"
  network = google_compute_network.main.name
  project = var.project_id

  direction = "INGRESS"
  priority  = 1000

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  source_ranges = ["::/0"]
  target_tags   = ["web-server"]
}

# Allow ICMPv6 for NDP and PMTUD
resource "google_compute_firewall" "allow_icmpv6" {
  name    = "allow-icmpv6"
  network = google_compute_network.main.name
  project = var.project_id

  direction = "INGRESS"
  priority  = 900

  allow {
    protocol = "icmpv6"
  }

  source_ranges = ["::/0"]
}

# Allow internal IPv6 traffic within VPC
resource "google_compute_firewall" "allow_internal_ipv6" {
  name    = "allow-internal-ipv6"
  network = google_compute_network.main.name
  project = var.project_id

  direction = "INGRESS"
  priority  = 1000

  allow {
    protocol = "all"
  }

  # ULA range for internal IPv6
  source_ranges = ["fd00::/8"]
}

# Deny all other IPv6 inbound (explicit deny)
resource "google_compute_firewall" "deny_all_ipv6" {
  name    = "deny-all-ipv6-ingress"
  network = google_compute_network.main.name
  project = var.project_id

  direction = "INGRESS"
  priority  = 65534

  deny {
    protocol = "all"
  }

  source_ranges = ["::/0"]
}
```

## Egress IPv6 Firewall Rules

```bash
# Allow all outbound IPv6 traffic from internal services
gcloud compute firewall-rules create allow-egress-ipv6 \
    --network="$VPC_NAME" \
    --direction=EGRESS \
    --priority=1000 \
    --destination-ranges="::/0" \
    --rules=all \
    --target-tags=outbound-allowed \
    --project="$PROJECT"

# Deny outbound to specific IPv6 prefix (blocklist)
gcloud compute firewall-rules create deny-egress-blocked-ipv6 \
    --network="$VPC_NAME" \
    --direction=EGRESS \
    --priority=500 \
    --destination-ranges="2001:db8:blocked::/48" \
    --rules=all \
    --project="$PROJECT"
```

## List and Audit IPv6 Firewall Rules

```bash
# List all firewall rules that apply to IPv6
gcloud compute firewall-rules list \
    --project="$PROJECT" \
    --format="table(name, direction, priority, sourceRanges, destinationRanges, allowed, targetTags)"

# Filter only rules with IPv6 source ranges
gcloud compute firewall-rules list \
    --project="$PROJECT" \
    --filter="sourceRanges:(*:*)" \
    --format="table(name, direction, priority, sourceRanges)"

# Describe a specific rule
gcloud compute firewall-rules describe allow-web-ipv6 \
    --project="$PROJECT"
```

## Conclusion

GCP VPC firewall rules for IPv6 use `::/0` for all IPv6 traffic ranges and the `icmpv6` protocol identifier for ICMPv6. Always allow ICMPv6 to ensure NDP (Neighbor Discovery) and Path MTU Discovery work correctly — blocking ICMPv6 breaks IPv6 connectivity. Use target-tags to scope rules to specific VM groups. Terraform's `google_compute_firewall` resource uses `source_ranges = ["::/0"]` for IPv6 ingress rules and `destination_ranges` for egress. Audit firewall rules regularly with `gcloud compute firewall-rules list --filter="sourceRanges:(*:*)"`.
