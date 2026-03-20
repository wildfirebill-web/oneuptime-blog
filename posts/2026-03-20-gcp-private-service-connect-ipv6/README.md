# How to Configure GCP Private Service Connect with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, Private Service Connect, PSC, Google Cloud, Private Endpoint

Description: Configure Google Cloud Private Service Connect (PSC) endpoints in dual-stack subnets for IPv6 access to Google APIs and published services, and set up PSC with IPv6 consumer endpoints.

## Introduction

Private Service Connect (PSC) allows private access to Google APIs and third-party services without using public IP addresses. PSC endpoints can be deployed in dual-stack subnets to support IPv6 access. When a PSC endpoint is created in a dual-stack subnet, it receives both an IPv4 and an IPv6 address, enabling IPv6 clients within the VPC to access Google APIs privately without traversing the internet.

## Create PSC Endpoint for Google APIs with IPv6

```bash
PROJECT="my-project"
REGION="us-east1"

# Ensure subnet is dual-stack
gcloud compute networks subnets describe subnet-private \
    --region="$REGION" \
    --project="$PROJECT" \
    --format="json(stackType)"

# Reserve static internal IPv4 address for PSC endpoint
gcloud compute addresses create psc-endpoint-ipv4 \
    --project="$PROJECT" \
    --region="$REGION" \
    --subnet=subnet-private \
    --purpose=PRIVATE_SERVICE_CONNECT

# Reserve static internal IPv6 address for PSC endpoint
gcloud compute addresses create psc-endpoint-ipv6 \
    --project="$PROJECT" \
    --region="$REGION" \
    --subnet=subnet-private \
    --ip-version=IPV6 \
    --purpose=PRIVATE_SERVICE_CONNECT

# Create PSC endpoint for all Google APIs
gcloud compute forwarding-rules create psc-google-apis \
    --project="$PROJECT" \
    --region="$REGION" \
    --network=vpc-main \
    --address=psc-endpoint-ipv4 \
    --target-google-apis-bundle=all-apis \
    --load-balancing-scheme=""

# Create PSC DNS entries for Google APIs
gcloud dns managed-zones create google-apis-private \
    --project="$PROJECT" \
    --dns-name="googleapis.com." \
    --description="Private DNS for Google APIs via PSC" \
    --visibility=private \
    --networks=vpc-main

# Add A record pointing to PSC endpoint
PSC_IP=$(gcloud compute addresses describe psc-endpoint-ipv4 \
    --region="$REGION" \
    --project="$PROJECT" \
    --format="get(address)")

gcloud dns record-sets create "*.googleapis.com." \
    --zone=google-apis-private \
    --type=A \
    --ttl=300 \
    --rrdatas="$PSC_IP" \
    --project="$PROJECT"
```

## Terraform PSC with IPv6 Subnet

```hcl
# psc_ipv6.tf

variable "project_id" {}
variable "region" { default = "us-east1" }

# Dual-stack subnet for PSC
resource "google_compute_subnetwork" "psc" {
  name          = "subnet-psc"
  ip_cidr_range = "10.0.20.0/24"
  region        = var.region
  network       = google_compute_network.main.id
  project       = var.project_id

  stack_type       = "IPV4_IPV6"
  ipv6_access_type = "INTERNAL"

  # Required for PSC consumer endpoints
  purpose = "PRIVATE"
}

# Static IPv4 address for PSC endpoint
resource "google_compute_address" "psc_ipv4" {
  name         = "psc-endpoint-ipv4"
  region       = var.region
  project      = var.project_id
  subnetwork   = google_compute_subnetwork.psc.id
  address_type = "INTERNAL"
  purpose      = "PRIVATE_SERVICE_CONNECT"
}

# PSC forwarding rule for Google APIs
resource "google_compute_forwarding_rule" "psc_google_apis" {
  name                    = "psc-google-apis"
  region                  = var.region
  project                 = var.project_id
  network                 = google_compute_network.main.id
  ip_address              = google_compute_address.psc_ipv4.id
  target                  = "all-apis"
  load_balancing_scheme   = ""
  allow_psc_global_access = true
}

# Private DNS zone for Google APIs
resource "google_dns_managed_zone" "google_apis" {
  name        = "google-apis-private"
  dns_name    = "googleapis.com."
  project     = var.project_id
  description = "Private DNS for PSC Google APIs access"
  visibility  = "private"

  private_visibility_config {
    networks {
      network_url = google_compute_network.main.id
    }
  }
}

# Wildcard A record for all APIs
resource "google_dns_record_set" "google_apis_wildcard" {
  name         = "*.googleapis.com."
  managed_zone = google_dns_managed_zone.google_apis.name
  project      = var.project_id
  type         = "A"
  ttl          = 300
  rrdatas      = [google_compute_address.psc_ipv4.address]
}
```

## Publish a Service via PSC with IPv6

```bash
# Create a service attachment for your own service
# Service is behind an internal load balancer

# Create PSC service attachment
gcloud compute service-attachments create my-service-psc \
    --project="$PROJECT" \
    --region="$REGION" \
    --forwarding-rules=my-ilb-rule \
    --connection-preference=ACCEPT_AUTOMATIC \
    --nat-subnets=subnet-psc-nat

# Consumer creates endpoint to access your service
gcloud compute forwarding-rules create consume-my-service \
    --project="$CONSUMER_PROJECT" \
    --region="$REGION" \
    --network=consumer-vpc \
    --address=consumer-psc-ip \
    --target=projects/$PROJECT/regions/$REGION/serviceAttachments/my-service-psc \
    --load-balancing-scheme=""

# Verify PSC connection is accepted
gcloud compute service-attachments describe my-service-psc \
    --region="$REGION" \
    --project="$PROJECT" \
    --format="json(connectedEndpoints)"
```

## Test PSC Connectivity over IPv6

```bash
# From a VM in the dual-stack subnet
# Test connectivity to Google APIs via PSC endpoint
gcloud compute ssh test-vm --project="$PROJECT" --zone=us-east1-b

# Inside VM, test IPv4 PSC
curl https://storage.googleapis.com/storage/v1/b \
    -H "Authorization: Bearer $(gcloud auth print-access-token)"

# Test connectivity (DNS resolves to PSC IPv4 endpoint)
dig storage.googleapis.com
# Returns: PSC endpoint IP (e.g., 10.0.20.100)

# For IPv6 access to PSC, IPv6 clients in the subnet use PSC DNS names
# The PSC endpoint handles protocol translation
```

## Conclusion

GCP Private Service Connect endpoints can be deployed in dual-stack subnets to provide private access to Google APIs. Create a static address with `purpose = "PRIVATE_SERVICE_CONNECT"` and a forwarding rule targeting `all-apis` or `vpc-sc`. Configure private DNS zones to resolve `*.googleapis.com` to the PSC endpoint IP. PSC endpoints receive connectivity from both IPv4 and IPv6 clients within the VPC. For outbound from IPv6-only VMs to Google APIs, ensure PSC is configured with appropriate DNS zones so all API calls route through the private endpoint.
