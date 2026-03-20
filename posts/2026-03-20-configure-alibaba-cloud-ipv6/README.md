# How to Configure Alibaba Cloud VPC with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Alibaba Cloud, VPC, Aliyun, Networking, Terraform

Description: Configure IPv6 on Alibaba Cloud VPC, enabling dual-stack networking on ECS instances with IPv6 internet access via IPv6 gateways.

## Introduction

Alibaba Cloud VPC supports IPv6 through a dual-stack model. IPv6 is enabled at the VPC level, then on VSwitch (subnet), and ECS instances. An IPv6 Gateway is required for IPv6 internet access, similar to a NAT Gateway for IPv4.

## Enabling IPv6 via Alibaba Cloud CLI

```bash
# Install Alibaba Cloud CLI

pip install aliyun-cli

# Configure credentials
aliyun configure

# Enable IPv6 on existing VPC
aliyun vpc ModifyVpcAttribute \
  --VpcId "vpc-..." \
  --EnableIPv6 true

# Create VSwitch with IPv6
aliyun vpc CreateVSwitch \
  --VpcId "vpc-..." \
  --ZoneId "cn-hangzhou-h" \
  --CidrBlock "192.168.1.0/24" \
  --Ipv6CidrBlock "0"    # 0 = first /64 from VPC's /56
```

## Terraform: Alibaba Cloud VPC with IPv6

```hcl
# provider.tf
provider "alicloud" {
  region     = "cn-hangzhou"
  access_key = var.access_key
  secret_key = var.secret_key
}

# VPC with IPv6 enabled
resource "alicloud_vpc" "main" {
  vpc_name    = "ipv6-vpc"
  cidr_block  = "192.168.0.0/16"
  enable_ipv6 = true

  # Alibaba Cloud auto-assigns a /56 IPv6 range to the VPC
}

# VSwitch (subnet) with IPv6 /64
resource "alicloud_vswitch" "public" {
  vpc_id               = alicloud_vpc.main.id
  cidr_block           = "192.168.1.0/24"
  zone_id              = "cn-hangzhou-h"
  vswitch_name         = "public-vswitch"
  ipv6_cidr_block_mask = 8  # 8th /64 block within the VPC /56
}

# ECS instance with dual-stack
resource "alicloud_instance" "web" {
  instance_name        = "web-server"
  image_id             = "ubuntu_22_04_x64_20G_alibase_20230208.vhd"
  instance_type        = "ecs.c6.large"
  vswitch_id           = alicloud_vswitch.public.id
  security_groups      = [alicloud_security_group.web.id]
  ipv6_address_count   = 1  # Assign 1 IPv6 address

  internet_max_bandwidth_out = 10
  internet_charge_type       = "PayByTraffic"
}
```

## IPv6 Gateway and Bandwidth

```hcl
# IPv6 Gateway (required for IPv6 internet access)
resource "alicloud_vpc_ipv6_gateway" "gw" {
  ipv6_gateway_name = "ipv6-gw"
  vpc_id            = alicloud_vpc.main.id
  spec              = "Small"  # Small/Medium/Large
}

# Assign public bandwidth to IPv6 addresses
resource "alicloud_vpc_ipv6_internet_bandwidth" "bw" {
  ipv6_address_id  = alicloud_instance.web.ipv6_addresses[0]
  internet_charge_type = "PayByBandwidth"
  bandwidth        = 100  # Mbps
  ipv6_gateway_id  = alicloud_vpc_ipv6_gateway.gw.id
}
```

## Security Group for IPv6

```hcl
resource "alicloud_security_group" "web" {
  name   = "web-sg"
  vpc_id = alicloud_vpc.main.id
}

# Allow IPv6 HTTP
resource "alicloud_security_group_rule" "http_ipv6" {
  type              = "ingress"
  ip_protocol       = "tcp"
  nic_type          = "intranet"
  policy            = "accept"
  port_range        = "80/80"
  priority          = 1
  security_group_id = alicloud_security_group.web.id
  cidr_ip           = "::/0"
  ipv6_cidr_ip      = "::/0"
}
```

## Verifying IPv6 on ECS

```bash
# SSH into instance (via IPv4 initially)
# Check IPv6 address assigned
ip -6 addr show eth0
# inet6 2408:4001:...::/128 scope global dynamic

# Test internet connectivity
ping6 2001:4860:4860::8888
curl -6 http://[2001:4860:4860::8888]/

# Verify routing
ip -6 route show
# ::/0 via fe80::1 dev eth0 proto ra
```

## Conclusion

Alibaba Cloud VPC IPv6 requires an IPv6 Gateway for internet access and explicit bandwidth assignment to each IPv6 address. Enable IPv6 at VPC creation and assign `/64` VSwitches with `ipv6_cidr_block_mask`. Use `alicloud_vpc_ipv6_internet_bandwidth` to control per-instance IPv6 egress bandwidth. Monitor Alibaba Cloud ECS instance IPv6 availability with OneUptime.
