# How to Use the cidrnetmask Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Networking

Description: Learn how to use the cidrnetmask function in OpenTofu to convert CIDR notation to subnet mask format for legacy systems and network device configurations.

## Introduction

The `cidrnetmask` function in OpenTofu converts a CIDR notation prefix to its equivalent subnet mask in dotted decimal notation. This is useful when configuring systems that require subnet masks in traditional format rather than CIDR notation.

## Syntax

```hcl
cidrnetmask(prefix)
```

- **prefix** - a CIDR notation block (e.g., `"10.0.0.0/24"`)
- Returns the subnet mask as a dotted decimal string (e.g., `"255.255.255.0"`)

## Basic Examples

```hcl
output "class_c_mask" {
  value = cidrnetmask("10.0.0.0/24")   # Returns "255.255.255.0"
}

output "class_b_mask" {
  value = cidrnetmask("10.0.0.0/16")   # Returns "255.255.0.0"
}

output "class_a_mask" {
  value = cidrnetmask("10.0.0.0/8")    # Returns "255.0.0.0"
}

output "slash_20" {
  value = cidrnetmask("10.0.0.0/20")   # Returns "255.255.240.0"
}
```

## Practical Use Cases

### VPN Gateway Configuration

```hcl
variable "vpn_network" {
  type    = string
  default = "10.100.0.0/22"
}

locals {
  vpn_mask = cidrnetmask(var.vpn_network)
}

resource "aws_customer_gateway" "on_prem" {
  bgp_asn    = 65000
  ip_address = "203.0.113.1"
  type       = "ipsec.1"

  tags = {
    Name    = "on-prem-gw"
    Network = var.vpn_network
    Mask    = local.vpn_mask
  }
}
```

### Network Device Configuration Templates

```hcl
variable "mgmt_subnet" {
  type    = string
  default = "172.16.0.0/24"
}

locals {
  mgmt_mask = cidrnetmask(var.mgmt_subnet)
  mgmt_net  = cidrhost(var.mgmt_subnet, 0)
}

locals {
  router_config = templatefile("${path.module}/templates/router.conf.tpl", {
    network = local.mgmt_net
    mask    = local.mgmt_mask
    gateway = cidrhost(var.mgmt_subnet, 1)
  })
}
```

### Generating ifcfg Network Scripts

```hcl
variable "interface_cidr" {
  type    = string
  default = "10.0.1.50/24"
}

locals {
  interface_mask = cidrnetmask(var.interface_cidr)
}

resource "local_file" "ifcfg" {
  content = <<-EOF
    TYPE=Ethernet
    BOOTPROTO=static
    IPADDR=${split("/", var.interface_cidr)[0]}
    NETMASK=${local.interface_mask}
    GATEWAY=${cidrhost(var.interface_cidr, 1)}
    ONBOOT=yes
  EOF
  filename = "${path.module}/scripts/ifcfg-eth0"
}
```

## Step-by-Step Usage

```bash
tofu console

> cidrnetmask("10.0.0.0/24")
"255.255.255.0"
> cidrnetmask("192.168.0.0/20")
"255.255.240.0"
```

## CIDR Prefix Length to Subnet Mask

| CIDR | Subnet Mask |
|------|-------------|
| /8 | 255.0.0.0 |
| /16 | 255.255.0.0 |
| /24 | 255.255.255.0 |
| /25 | 255.255.255.128 |
| /26 | 255.255.255.192 |
| /28 | 255.255.255.240 |

## Conclusion

The `cidrnetmask` function bridges the gap between CIDR notation (used in OpenTofu networking) and traditional subnet mask notation (required by legacy systems and network device configurations). Use it when generating network device configurations, VPN templates, or any system that expects subnet masks in dotted decimal format.
