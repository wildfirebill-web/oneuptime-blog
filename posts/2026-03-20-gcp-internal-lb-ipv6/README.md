# How to Configure GCP Internal Load Balancer with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, Internal Load Balancer, Private Networking, Cloud

Description: A guide to configuring GCP Internal HTTP(S) Load Balancer and Internal TCP/UDP Load Balancer with IPv6 for private network traffic within GCP.

GCP Internal Load Balancer enables IPv6 for traffic within your VPC, useful for microservices communication, internal APIs, and cross-region private connectivity. Internal IPv6 load balancing requires a dual-stack subnet and uses private IPv6 addresses.

## Prerequisites

```bash
# Enable required APIs

gcloud services enable compute.googleapis.com

# Verify your VPC subnet supports IPv6
gcloud compute networks subnets describe my-subnet \
  --region=us-east1 \
  --format="value(ipv6CidrRange,stackType)"
```

## Create Dual-Stack Subnet for Internal IPv6

```bash
# Create or update subnet to dual-stack
gcloud compute networks subnets update my-subnet \
  --region=us-east1 \
  --stack-type=IPV4_IPV6 \
  --ipv6-access-type=INTERNAL

# Or create a new dual-stack subnet
gcloud compute networks subnets create internal-subnet \
  --network=my-vpc \
  --region=us-east1 \
  --range=10.0.1.0/24 \
  --stack-type=IPV4_IPV6 \
  --ipv6-access-type=INTERNAL
```

## Terraform: Internal TCP/UDP LB with IPv6

```hcl
# Internal forwarding rule with IPv6
resource "google_compute_address" "internal_ipv6" {
  name         = "internal-lb-ipv6"
  address_type = "INTERNAL"
  ip_version   = "IPV6"
  subnetwork   = google_compute_subnetwork.main.id
  region       = "us-east1"
}

resource "google_compute_forwarding_rule" "internal_ipv6" {
  name                  = "internal-lb-fwd-ipv6"
  region                = "us-east1"
  load_balancing_scheme = "INTERNAL"
  backend_service       = google_compute_region_backend_service.main.id
  ip_address            = google_compute_address.internal_ipv6.id
  ip_protocol           = "TCP"
  ports                 = ["80", "443"]
  network               = google_compute_network.main.id
  subnetwork            = google_compute_subnetwork.main.id
}

resource "google_compute_region_backend_service" "main" {
  name                  = "internal-backend"
  region                = "us-east1"
  load_balancing_scheme = "INTERNAL"
  protocol              = "TCP"

  backend {
    group = google_compute_instance_group.main.id
  }

  health_checks = [google_compute_region_health_check.main.id]
}

resource "google_compute_region_health_check" "main" {
  name   = "internal-health-check"
  region = "us-east1"

  tcp_health_check {
    port = 80
  }
}
```

## Internal HTTP(S) Load Balancer with IPv6

```hcl
# Internal HTTP LB - proxy-based, supports IPv6 frontend
resource "google_compute_address" "internal_http_ipv6" {
  name         = "internal-http-lb-ipv6"
  address_type = "INTERNAL"
  ip_version   = "IPV6"
  purpose      = "SHARED_LOADBALANCER_VIP"
  subnetwork   = google_compute_subnetwork.proxy_subnet.id
  region       = "us-east1"
}

resource "google_compute_subnetwork" "proxy_subnet" {
  name          = "proxy-subnet"
  ip_cidr_range = "10.10.0.0/24"
  region        = "us-east1"
  network       = google_compute_network.main.id
  purpose       = "REGIONAL_MANAGED_PROXY"
  role          = "ACTIVE"
  stack_type    = "IPV4_IPV6"
}
```

## gcloud CLI: Internal IPv6 LB

```bash
# Reserve internal IPv6 address
gcloud compute addresses create internal-lb-ipv6 \
  --region=us-east1 \
  --subnet=my-subnet \
  --ip-version=IPV6 \
  --address-type=INTERNAL

# Create internal forwarding rule
gcloud compute forwarding-rules create internal-lb-fwd-ipv6 \
  --region=us-east1 \
  --load-balancing-scheme=INTERNAL \
  --backend-service=internal-backend \
  --address=internal-lb-ipv6 \
  --ports=80

# Verify internal IPv6 address
gcloud compute addresses describe internal-lb-ipv6 --region=us-east1
```

## Firewall Rules for Internal IPv6

```bash
# Allow traffic to backends from internal IPv6 range
gcloud compute firewall-rules create allow-internal-ipv6 \
  --network=my-vpc \
  --allow=tcp:80,tcp:443 \
  --source-ranges="fd20::/20"   # GCP internal IPv6 ranges

# Allow health check probes
gcloud compute firewall-rules create allow-health-check-ipv6 \
  --network=my-vpc \
  --allow=tcp \
  --source-ranges="2600:2d00:1:b029::/64"   # GCP health check IPv6 range
```

## Accessing Internal IPv6 LB

```bash
# From a GCP instance in the same VPC
curl -6 http://[fd20::internal-lb-address]/

# Using internal DNS if configured
curl -6 http://internal-service.internal.example.com/
```

GCP Internal Load Balancer's IPv6 support enables modern microservice architectures where internal services communicate over IPv6, reducing dependency on IPv4 private address space management within large GCP deployments.
