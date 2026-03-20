# How to Configure IPv6 on Google Compute Engine VMs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, Compute Engine, VM, Dual-Stack, Google Cloud

Description: Assign IPv6 addresses to Google Compute Engine VM instances, configure dual-stack network interfaces, and verify IPv6 connectivity inside GCE VMs.

## Introduction

Google Compute Engine VMs receive IPv6 addresses when placed in dual-stack subnets. Each VM's network interface can have both an IPv4 address and an IPv6 address. External IPv6 VMs get globally routable addresses, while internal IPv6 VMs get ULA addresses. IPv6 configuration in GCE is automatic - once the subnet supports IPv6, VMs launched in it receive IPv6 addresses.

## Create VM with IPv6 Address

```bash
PROJECT="my-project"
ZONE="us-east1-b"

# Create VM in dual-stack subnet (gets IPv6 automatically)

gcloud compute instances create vm-web-01 \
    --project="$PROJECT" \
    --zone="$ZONE" \
    --machine-type=n2-standard-2 \
    --subnet=subnet-web \
    --network-interface=subnet=subnet-web,stack-type=IPV4_IPV6,ipv6-network-tier=PREMIUM \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --boot-disk-size=20GB

# Specify specific IPv6 address (optional)
gcloud compute instances create vm-web-static \
    --project="$PROJECT" \
    --zone="$ZONE" \
    --machine-type=n2-standard-2 \
    --network-interface=subnet=subnet-web,stack-type=IPV4_IPV6,ipv6-address=2600:1900::/128 \
    --image-family=debian-12 \
    --image-project=debian-cloud
```

## Add IPv6 to Existing VM

```bash
# Update existing network interface to enable IPv6
gcloud compute instances network-interfaces update vm-existing \
    --project="$PROJECT" \
    --zone="$ZONE" \
    --network-interface=nic0 \
    --stack-type=IPV4_IPV6 \
    --ipv6-network-tier=PREMIUM

# Restart the VM for changes to take effect
gcloud compute instances stop vm-existing --zone="$ZONE"
gcloud compute instances start vm-existing --zone="$ZONE"
```

## Terraform GCE VM with IPv6

```hcl
# gce_vm_ipv6.tf

resource "google_compute_instance" "web" {
  name         = "vm-web-01"
  machine_type = "n2-standard-2"
  zone         = "us-east1-b"
  project      = var.project_id

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 20
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.web.self_link

    # Enable dual-stack IPv6
    stack_type         = "IPV4_IPV6"
    ipv6_network_tier  = "PREMIUM"

    # IPv4 access config (for external IPv4)
    access_config {
      network_tier = "PREMIUM"
    }
  }

  metadata = {
    enable-oslogin = "TRUE"
  }

  tags = ["web-server"]

  labels = {
    environment = "production"
  }
}

output "vm_ipv4" {
  value = google_compute_instance.web.network_interface[0].access_config[0].nat_ip
}

output "vm_ipv6" {
  value = google_compute_instance.web.network_interface[0].ipv6_address
}
```

## Verify IPv6 Inside the VM

```bash
# SSH into the VM
gcloud compute ssh vm-web-01 \
    --project="$PROJECT" \
    --zone="$ZONE"

# Inside VM, check IPv6 address
ip -6 addr show

# Expected: IPv6 address on ens4 interface
# 2600:1900:4000:abc1:8000:: for external
# fd20:0000:0000:0001:: for internal

# Test IPv6 connectivity
ping6 -c 3 2001:4860:4860::8888  # Google DNS
curl -6 -s https://ipv6.icanhazip.com

# Test DNS resolution
dig AAAA google.com
```

## External IPv6 vs Internal IPv6 VMs

```bash
# Check which type of IPv6 a VM has
gcloud compute instances describe vm-web-01 \
    --project="$PROJECT" \
    --zone="$ZONE" \
    --format="json(networkInterfaces[].{ipv6Access:ipv6AccessType, ipv6Addr:ipv6Address})"

# External IPv6 VM:
# Can receive inbound connections from internet (if firewall allows)
# ipv6AccessType: EXTERNAL

# Internal IPv6 VM:
# Only reachable within VPC
# Can reach internet via Cloud NAT for IPv6
# ipv6AccessType: INTERNAL

# Firewall rule needed to allow inbound to external IPv6 VM
gcloud compute firewall-rules create allow-http-ipv6 \
    --network=vpc-main \
    --direction=INGRESS \
    --source-ranges="::/0" \
    --rules=tcp:80,tcp:443 \
    --target-tags=web-server
```

## Instance Templates with IPv6

```bash
# Create instance template with IPv6 support
gcloud compute instance-templates create tmpl-web-ipv6 \
    --project="$PROJECT" \
    --machine-type=n2-standard-2 \
    --subnet=subnet-web \
    --network-interface=subnet=projects/$PROJECT/regions/us-east1/subnetworks/subnet-web,stack-type=IPV4_IPV6,ipv6-network-tier=PREMIUM \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --tags=web-server

# Use template for managed instance group
gcloud compute instance-groups managed create mig-web \
    --project="$PROJECT" \
    --base-instance-name=web \
    --template=tmpl-web-ipv6 \
    --size=3 \
    --zone="$ZONE"
```

## Conclusion

GCE VMs in dual-stack subnets automatically receive IPv6 addresses through their network interface. Configure `stack_type = "IPV4_IPV6"` in the network interface definition in Terraform or use `--stack-type=IPV4_IPV6` with gcloud. External IPv6 VMs are reachable from the internet when firewall rules permit; internal IPv6 VMs use Cloud NAT for outbound internet access. Check VM IPv6 addresses with `ip -6 addr show` inside the instance or via `gcloud compute instances describe`.
