# How to Configure IBM Cloud VPC with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IBM Cloud, VPC, Networking, Terraform

Description: Configure IPv6 on IBM Cloud VPC infrastructure, including enabling IPv6 on subnets and instances, and configuring security groups for IPv6 traffic.

## Introduction

IBM Cloud VPC supports IPv6 addressing. IPv6 can be enabled on VPC subnets, and virtual server instances receive both IPv4 and IPv6 addresses. IBM Cloud provides dual-stack support across its VPC network fabric.

## Creating a VPC with IPv6 Subnet

```bash
# Install IBM Cloud CLI with VPC plugin

ibmcloud plugin install vpc-infrastructure

# Create VPC
ibmcloud is vpc-create my-vpc --resource-group-name default

# Create subnet with IPv6
ibmcloud is subnet-create my-subnet \
  --vpc my-vpc \
  --zone us-south-1 \
  --ipv4-cidr-block 10.0.0.0/24 \
  --ipv6-cidr-block "2607:f0d0:10::/48"
```

## Terraform: IBM Cloud VPC with IPv6

```hcl
# provider.tf
terraform {
  required_providers {
    ibm = {
      source = "IBM-Cloud/ibm"
    }
  }
}

provider "ibm" {
  region = "us-south"
}

# VPC
resource "ibm_is_vpc" "main" {
  name = "ipv6-vpc"
}

# Subnet with IPv6
resource "ibm_is_subnet" "public" {
  name            = "public-subnet"
  vpc             = ibm_is_vpc.main.id
  zone            = "us-south-1"
  ipv4_cidr_block = "10.0.1.0/24"
}

# Instance with IPv6
resource "ibm_is_instance" "web" {
  name    = "web-server"
  vpc     = ibm_is_vpc.main.id
  zone    = "us-south-1"
  image   = data.ibm_is_image.ubuntu.id
  profile = "cx2-2x4"

  primary_network_interface {
    subnet = ibm_is_subnet.public.id
  }

  keys = [data.ibm_is_ssh_key.my_key.id]
}

data "ibm_is_image" "ubuntu" {
  name = "ibm-ubuntu-22-04-3-minimal-amd64-3"
}
```

## Security Group for IPv6

```hcl
resource "ibm_is_security_group" "web" {
  name = "web-sg"
  vpc  = ibm_is_vpc.main.id
}

# Allow inbound HTTP from IPv6
resource "ibm_is_security_group_rule" "http_ipv6" {
  group     = ibm_is_security_group.web.id
  direction = "inbound"
  remote    = "::/0"

  tcp {
    port_min = 80
    port_max = 80
  }
}

# Allow inbound HTTPS from IPv6
resource "ibm_is_security_group_rule" "https_ipv6" {
  group     = ibm_is_security_group.web.id
  direction = "inbound"
  remote    = "::/0"

  tcp {
    port_min = 443
    port_max = 443
  }
}

# Allow ICMPv6
resource "ibm_is_security_group_rule" "icmpv6" {
  group     = ibm_is_security_group.web.id
  direction = "inbound"
  remote    = "::/0"
  icmp {
    type = -1
    code = -1
  }
}
```

## Floating IP (Public IPv6)

```hcl
# Assign a floating IPv6 to the instance
resource "ibm_is_floating_ip" "web" {
  name   = "web-floating-ip"
  target = ibm_is_instance.web.primary_network_interface[0].id
}

output "public_ipv6" {
  value = ibm_is_floating_ip.web.address
}
```

## Testing Connectivity

```bash
# SSH to instance
ssh root@$(terraform output -raw public_ipv6)

# Verify IPv6 inside instance
ip -6 addr show
ip -6 route show default

# Test outbound IPv6
ping6 2001:4860:4860::8888
curl -6 https://ipv6.icanhazip.com

# Test inbound (from external host)
curl -6 https://[<floating-ipv6>]/
```

## Conclusion

IBM Cloud VPC supports IPv6 on subnets and instances with floating IPv6 addresses for external access. Configure security groups with `::/0` rules for IPv6 ingress and egress. Use Terraform with the `IBM-Cloud/ibm` provider for Infrastructure as Code deployments. Monitor IBM Cloud instance IPv6 health and latency with OneUptime.
