# How to Configure Hetzner Cloud IPv6 with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Hetzner Cloud, Terraform, IPv6, Server, Networking, Cloud

Description: A guide to provisioning Hetzner Cloud servers and networks with IPv6 addressing using Terraform, including floating IP and DNS configuration.

Hetzner Cloud provides a free /64 IPv6 network prefix for every server. Each server's primary network interface automatically receives an IPv6 address. Hetzner also supports IPv6 Floating IPs and private networks. This guide covers common IPv6 configurations with Terraform.

## Step 1: Configure the Hetzner Cloud Provider

```hcl
# provider.tf

terraform {
  required_providers {
    hcloud = {
      source  = "hetznercloud/hcloud"
      version = "~> 1.44"
    }
  }
}

provider "hcloud" {
  # Set HCLOUD_TOKEN env var or use token attribute
  token = var.hcloud_token
}

variable "hcloud_token" {
  type      = string
  sensitive = true
}
```

## Step 2: Create a Server (IPv6 Is Automatic)

Hetzner Cloud servers automatically receive a /64 IPv6 prefix. No explicit IPv6 flag is needed:

```hcl
# server.tf - Hetzner Cloud server with automatic IPv6
resource "hcloud_server" "web" {
  name        = "web-01"
  image       = "ubuntu-22.04"
  server_type = "cx22"
  location    = "nbg1"

  # SSH key for access
  ssh_keys = [hcloud_ssh_key.main.id]

  # Optional: user_data to configure additional IPv6 settings
  user_data = <<-EOF
    #cloud-config
    package_update: true
    packages:
      - curl
      - iputils-ping
  EOF

  labels = {
    role = "web"
    ipv6 = "enabled"
  }
}

output "server_ipv4" {
  value = hcloud_server.web.ipv4_address
}

output "server_ipv6" {
  value = hcloud_server.web.ipv6_address
  description = "The primary IPv6 address (/64 block)"
}

output "server_ipv6_network" {
  value = hcloud_server.web.ipv6_network
  description = "The full /64 prefix assigned to this server"
}
```

## Step 3: Create an IPv6 Floating IP

Floating IPs in Hetzner can be IPv4 or IPv6. IPv6 Floating IPs are /64 prefixes that can be reassigned between servers:

```hcl
# floating-ip.tf - Create an IPv6 Floating IP
resource "hcloud_floating_ip" "web_ipv6" {
  type          = "ipv6"
  home_location = "nbg1"
  description   = "Primary IPv6 floating IP for web tier"

  labels = {
    environment = "production"
  }
}

# Assign the floating IP to the server
resource "hcloud_floating_ip_assignment" "web" {
  floating_ip_id = hcloud_floating_ip.web_ipv6.id
  server_id      = hcloud_server.web.id
}

output "floating_ipv6" {
  value = hcloud_floating_ip.web_ipv6.ip_address
}
```

## Step 4: Configure the OS to Use the Floating IPv6

After assigning the floating IP, configure the OS to bind it. Add this to user_data or run via remote-exec:

```bash
# On the server: configure the floating IPv6 on the interface
# Hetzner floating IPs must be configured in the OS manually

# Add the floating IPv6 address to eth0 (replace with your address)
ip -6 addr add 2a01:4f8:1:2::1/128 dev eth0

# Make it persistent (Netplan example for Ubuntu)
cat > /etc/netplan/60-floating-ipv6.yaml <<'NETPLAN'
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 2a01:4f8:1:2::1/128
NETPLAN
netplan apply
```

## Step 5: Add RDNS (Reverse DNS) for IPv6

```hcl
# rdns.tf - Set reverse DNS for the server's IPv6 address
resource "hcloud_rdns" "web_ipv6" {
  server_id  = hcloud_server.web.id
  ip_address = hcloud_server.web.ipv6_address
  dns_ptr    = "web-01.example.com"
}
```

## Step 6: Apply and Test

```bash
terraform apply

# Test SSH over IPv6
SERVER_IPV6=$(terraform output -raw server_ipv6)
ssh root@"$SERVER_IPV6"

# Test outbound IPv6 from the server
ssh root@"$SERVER_IPV6" 'ping6 -c 3 ipv6.google.com'

# Test RDNS
dig -x "$SERVER_IPV6"
```

Hetzner Cloud's automatic IPv6 assignment and competitive pricing make it an excellent platform for running IPv6-native or dual-stack workloads at low cost.
