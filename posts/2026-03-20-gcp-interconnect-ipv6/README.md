# How to Configure GCP Interconnect with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, Interconnect, Cloud Interconnect, BGP, Dual-Stack, Hybrid Cloud

Description: Configure Google Cloud Dedicated Interconnect and Partner Interconnect VLAN attachments for IPv6, enable BGP sessions for IPv6 route advertisement, and test dual-stack hybrid connectivity.

## Introduction

Google Cloud Interconnect supports IPv6 on VLAN attachments, enabling dual-stack connectivity between on-premises networks and GCP VPCs. IPv6 over Interconnect requires configuring BGP sessions for IPv6 route exchange. The BGP session itself uses IPv4 or link-local IPv6 addresses, while IPv6 prefixes are exchanged as IPv6 BGP address family (AFI/SAFI 2/1). Both Dedicated Interconnect and Partner Interconnect support IPv6.

## Configure VLAN Attachment with IPv6

```bash
PROJECT="my-project"
REGION="us-east1"

# Create Cloud Router for Interconnect
gcloud compute routers create router-interconnect \
    --project="$PROJECT" \
    --network=vpc-main \
    --region="$REGION" \
    --asn=65000

# Create Interconnect VLAN attachment with IPv6
gcloud compute interconnects attachments dedicated create vlan-attachment-1 \
    --project="$PROJECT" \
    --region="$REGION" \
    --router=router-interconnect \
    --interconnect=my-interconnect \
    --vlan=100 \
    --bandwidth=10g \
    --stack-type=IPV4_IPV6

# View the attachment configuration
gcloud compute interconnects attachments describe vlan-attachment-1 \
    --project="$PROJECT" \
    --region="$REGION" \
    --format="json(stackType, cloudRouterIpv6Address, customerRouterIpv6Address)"

# The output shows:
# cloudRouterIpv6Address: link-local or GCP-assigned IPv6 for BGP
# customerRouterIpv6Address: IPv6 to configure on your on-prem router
```

## Configure BGP for IPv6 Route Exchange

```bash
# After creating VLAN attachment, get BGP peer info
gcloud compute routers describe router-interconnect \
    --region="$REGION" \
    --project="$PROJECT" \
    --format="json(bgpPeers)"

# Update Cloud Router BGP peer to enable IPv6
gcloud compute routers update-bgp-peer router-interconnect \
    --project="$PROJECT" \
    --region="$REGION" \
    --peer-name=bgp-peer-1 \
    --enable-ipv6 \
    --ipv6-nexthop-address=<link-local-ipv6>

# Advertise IPv6 prefixes to on-premises via BGP
gcloud compute routers update router-interconnect \
    --project="$PROJECT" \
    --region="$REGION" \
    --advertisement-mode=CUSTOM \
    --set-advertisement-ranges="10.0.0.0/16,2600:1900:4000::/48"

# View BGP session status including IPv6
gcloud compute routers get-status router-interconnect \
    --project="$PROJECT" \
    --region="$REGION" \
    --format="json(result.bgpPeerStatus)"
```

## Terraform Interconnect with IPv6

```hcl
# interconnect_ipv6.tf

variable "project_id" {}
variable "region" { default = "us-east1" }

# Cloud Router for Interconnect
resource "google_compute_router" "interconnect" {
  name    = "router-interconnect"
  region  = var.region
  network = google_compute_network.main.id
  project = var.project_id

  bgp {
    asn               = 65000
    advertise_mode    = "CUSTOM"
    advertised_groups = ["ALL_SUBNETS"]

    # Advertise IPv6 prefix
    advertised_ip_ranges {
      range = "2600:1900:4000::/48"
    }
  }
}

# Dedicated Interconnect VLAN attachment with IPv6
resource "google_compute_interconnect_attachment" "vlan_1" {
  name         = "vlan-attachment-1"
  region       = var.region
  project      = var.project_id
  type         = "DEDICATED"
  interconnect = "projects/${var.project_id}/global/interconnects/my-interconnect"
  router       = google_compute_router.interconnect.id
  vlan_tag8021q = 100
  bandwidth    = "BPS_10G"

  # Enable dual-stack on the attachment
  stack_type = "IPV4_IPV6"
}

# Output BGP peer addresses for on-prem configuration
output "cloud_router_ipv6" {
  value = google_compute_interconnect_attachment.vlan_1.cloud_router_ipv6_interface_id
}

output "customer_router_ipv6" {
  value = google_compute_interconnect_attachment.vlan_1.customer_router_ipv6_interface_id
}
```

## On-Premises Router Configuration (Example: Cisco)

```
! On-premises router configuration for IPv6 BGP over Interconnect
! Interface configuration for VLAN 100
interface GigabitEthernet0/0.100
  encapsulation dot1q 100
  ip address 169.254.1.2 255.255.255.252
  ipv6 address <customer-router-ipv6-address>
  ipv6 enable

! BGP configuration with IPv6 address family
router bgp 65001
  neighbor 169.254.1.1 remote-as 65000
  neighbor 169.254.1.1 description GCP Cloud Router
  !
  address-family ipv6
    neighbor 169.254.1.1 activate
    network 2001:db8:onprem::/48
  exit-address-family
```

## Verify IPv6 Interconnect Connectivity

```bash
# Check BGP session status
gcloud compute routers get-status router-interconnect \
    --project="$PROJECT" \
    --region="$REGION" \
    --format="table(result.bgpPeerStatus[].name, result.bgpPeerStatus[].status, result.bgpPeerStatus[].numLearnedRoutes)"

# Check IPv6 routes learned from on-premises
gcloud compute routes list \
    --project="$PROJECT" \
    --filter="network=vpc-main" \
    --format="table(name, destRange, nextHopVpnTunnel, priority)"

# Test IPv6 connectivity from GCP VM to on-premises
gcloud compute ssh test-vm --project="$PROJECT" --zone=us-east1-b
# Inside VM:
ping6 -c 3 2001:db8:onprem::1  # On-premises IPv6 host
traceroute6 2001:db8:onprem::1
```

## Conclusion

GCP Interconnect VLAN attachments support IPv6 by setting `stack_type = "IPV4_IPV6"` on the attachment resource. BGP sessions can exchange IPv6 prefixes using the IPv6 address family, and Cloud Router BGP peers can be updated with `--enable-ipv6` to activate IPv6 route advertisement. Configure on-premises routers with corresponding BGP IPv6 address-family settings. Verify connectivity with `gcloud compute routers get-status` and ping6 tests from GCP VMs to on-premises IPv6 addresses.
