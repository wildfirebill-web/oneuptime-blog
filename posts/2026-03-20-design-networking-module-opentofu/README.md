# How to Design a Networking Module for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Networking, VPC, Transit Gateway, AWS, Module

Description: Learn how to design a comprehensive networking module for OpenTofu that manages VPC peering, Transit Gateway attachments, and cross-account network connectivity.

## Introduction

A networking module sits above the VPC module and handles inter-VPC connectivity: peering connections, Transit Gateway attachments, and route propagation. This is the glue that connects multiple VPCs in a hub-and-spoke or mesh architecture.

## variables.tf

```hcl
variable "name"        { type = string }
variable "environment" { type = string }

# VPC peering connections to establish

variable "vpc_peering_connections" {
  type = map(object({
    requester_vpc_id  = string
    accepter_vpc_id   = string
    accepter_region   = optional(string)
    accepter_owner_id = optional(string)
    auto_accept       = optional(bool, true)
    routes = list(object({
      requester_route_table_id = string
      accepter_route_table_id  = string
      requester_cidr           = string
      accepter_cidr            = string
    }))
  }))
  default = {}
}

# Transit Gateway attachments
variable "transit_gateway_id" { type = string; default = "" }
variable "tgw_attachments" {
  type = map(object({
    vpc_id      = string
    subnet_ids  = list(string)
    routes = list(object({
      route_table_id = string
      cidr_block     = string
    }))
  }))
  default = {}
}

variable "tags" { type = map(string); default = {} }
```

## main.tf

```hcl
locals {
  tags = merge({ Environment = var.environment, ManagedBy = "OpenTofu" }, var.tags)
}

# VPC Peering Connections
resource "aws_vpc_peering_connection" "connections" {
  for_each = var.vpc_peering_connections

  vpc_id        = each.value.requester_vpc_id
  peer_vpc_id   = each.value.accepter_vpc_id
  peer_region   = each.value.accepter_region
  peer_owner_id = each.value.accepter_owner_id
  auto_accept   = each.value.accepter_region == null ? each.value.auto_accept : false

  tags = merge(local.tags, { Name = "${var.name}-peering-${each.key}" })
}

# Accept cross-region peering connections (in the accepter region)
resource "aws_vpc_peering_connection_accepter" "cross_region" {
  for_each = {
    for k, v in var.vpc_peering_connections : k => v
    if v.accepter_region != null && v.accepter_region != data.aws_region.current.name
  }

  vpc_peering_connection_id = aws_vpc_peering_connection.connections[each.key].id
  auto_accept               = true
  tags                      = merge(local.tags, { Name = "${var.name}-peering-accepter-${each.key}" })
}

# Routes for peering connections
locals {
  peering_routes = flatten([
    for conn_name, conn in var.vpc_peering_connections : [
      for route in conn.routes : {
        key                       = "${conn_name}-requester"
        route_table_id            = route.requester_route_table_id
        destination_cidr          = route.accepter_cidr
        vpc_peering_connection_id = aws_vpc_peering_connection.connections[conn_name].id
      }
    ]
  ])
}

resource "aws_route" "peering" {
  for_each = { for r in local.peering_routes : r.key => r }

  route_table_id            = each.value.route_table_id
  destination_cidr_block    = each.value.destination_cidr
  vpc_peering_connection_id = each.value.vpc_peering_connection_id
}

# Transit Gateway Attachments
resource "aws_ec2_transit_gateway_vpc_attachment" "attachments" {
  for_each = var.tgw_attachments

  transit_gateway_id = var.transit_gateway_id
  vpc_id             = each.value.vpc_id
  subnet_ids         = each.value.subnet_ids
  tags               = merge(local.tags, { Name = "${var.name}-tgw-${each.key}" })
}

locals {
  tgw_routes = flatten([
    for att_name, att in var.tgw_attachments : [
      for route in att.routes : {
        key               = "${att_name}-${route.cidr_block}"
        route_table_id    = route.route_table_id
        cidr_block        = route.cidr_block
        attachment_name   = att_name
      }
    ]
  ])
}

resource "aws_route" "tgw" {
  for_each = { for r in local.tgw_routes : r.key => r }

  route_table_id         = each.value.route_table_id
  destination_cidr_block = each.value.cidr_block
  transit_gateway_id     = var.transit_gateway_id
}

data "aws_region" "current" {}
```

## Conclusion

This networking module handles both VPC peering and Transit Gateway connectivity patterns. The flattened route construction handles the many-to-many relationship between connections and route tables elegantly. Use this module to connect hub-and-spoke VPC architectures without managing peering and route configurations manually across multiple state files.
