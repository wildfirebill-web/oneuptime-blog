# How to Configure Linode IPv6 with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linode, Akamai Cloud, Terraform, IPv6, Networking, Cloud

Description: A guide to provisioning Linode (Akamai Cloud) instances with IPv6 addressing and managing DNS AAAA records using Terraform.

Linode (now Akamai Cloud) automatically assigns a /128 IPv6 address (SLAAC) to every Linode instance from a shared /64 prefix. Additional IPv6 ranges (/56 or /64) can be requested per region. This guide covers IPv6 configuration with the Linode Terraform provider.

## Step 1: Configure the Linode Provider

```hcl
# provider.tf
terraform {
  required_providers {
    linode = {
      source  = "linode/linode"
      version = "~> 2.0"
    }
  }
}

provider "linode" {
  # Set LINODE_TOKEN env var
  token = var.linode_token
}

variable "linode_token" {
  type      = string
  sensitive = true
}
```

## Step 2: Create a Linode Instance (IPv6 Is Automatic)

All Linode instances get a SLAAC IPv6 address automatically:

```hcl
# linode.tf - Linode instance with automatic IPv6
resource "linode_instance" "web" {
  label      = "web-ipv6"
  image      = "linode/ubuntu22.04"
  region     = "us-east"
  type       = "g6-nanode-1"

  # Root password (use SSH keys in production)
  root_pass  = var.root_password

  # Optional: SSH key authorization
  authorized_keys = [var.ssh_public_key]

  tags = ["web", "ipv6"]
}

output "instance_ipv4" {
  value = linode_instance.web.ip_address
}

output "instance_ipv6" {
  value = linode_instance.web.ipv6
  description = "SLAAC-assigned IPv6 address of the instance"
}
```

## Step 3: Request a Dedicated IPv6 Range

Linode allows requesting a dedicated /56 or /64 IPv6 range for more flexible addressing:

```hcl
# ipv6-range.tf - Request a dedicated IPv6 range for the instance
resource "linode_ipv6_range" "dedicated" {
  prefix_length = 64   # Request a /64 prefix
  linode_id     = linode_instance.web.id
  region        = linode_instance.web.region

  # route_target specifies which Linode's SLAAC address routes the prefix
  route_target  = linode_instance.web.ipv6
}

output "ipv6_range" {
  value = linode_ipv6_range.dedicated.range
  description = "Dedicated /64 IPv6 range assigned to this Linode"
}
```

## Step 4: Configure the OS to Use the Dedicated Range

After Terraform provisions the range, configure the instance to bind a specific address from it:

```hcl
# Use remote-exec to add the IPv6 address to the interface
resource "null_resource" "configure_ipv6" {
  depends_on = [linode_ipv6_range.dedicated]

  connection {
    type     = "ssh"
    host     = linode_instance.web.ip_address
    user     = "root"
    password = var.root_password
  }

  provisioner "remote-exec" {
    inline = [
      # Extract the first address from the /64 range and configure it
      "IPV6_RANGE=${linode_ipv6_range.dedicated.range}",
      "IPV6_ADDR=$(echo $IPV6_RANGE | sed 's|/64|1/64|')",
      "ip -6 addr add $IPV6_ADDR dev eth0",
      "echo \"ipv6_addr=$IPV6_ADDR\" >> /etc/environment"
    ]
  }
}
```

## Step 5: Create AAAA DNS Records

```hcl
# dns.tf - Create a Linode DNS domain and AAAA record
resource "linode_domain" "main" {
  type      = "master"
  domain    = "example.com"
  soa_email = "admin@example.com"
}

resource "linode_domain_record" "web_aaaa" {
  domain_id   = linode_domain.main.id
  name        = "web"
  record_type = "AAAA"
  target      = linode_instance.web.ipv6
  ttl_sec     = 300
}
```

## Step 6: Apply and Verify

```bash
terraform apply

# Test IPv6 connectivity to the instance
WEB_IPV6=$(terraform output -raw instance_ipv6 | cut -d'/' -f1)
ping6 -c 3 "$WEB_IPV6"

# SSH over IPv6
ssh root@"$WEB_IPV6"

# Test outbound IPv6
ssh root@"$WEB_IPV6" 'curl -6 https://ipv6.icanhazip.com'
```

Linode's automatic SLAAC addressing ensures every instance is immediately reachable over IPv6, while dedicated range requests allow for predictable addressing in production environments.
