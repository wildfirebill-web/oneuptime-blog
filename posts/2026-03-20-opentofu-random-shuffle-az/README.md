# How to Use Random Shuffle for Availability Zone Selection in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Random, Availability Zones, Random_shuffle, Infrastructure Distribution

Description: Learn how to use random_shuffle in OpenTofu for randomized availability zone assignment to distribute resources across zones without hardcoding zone names.

## Overview

`random_shuffle` randomizes a list, enabling even distribution of resources across availability zones without hardcoding. This is useful for assigning specific resources to zones in a randomized but stable manner.

## Step 1: Basic random_shuffle for AZ Selection

```hcl
# main.tf - Randomized AZ selection

data "aws_availability_zones" "available" {
  state = "available"
}

# Shuffle available AZs for random distribution
resource "random_shuffle" "az_shuffle" {
  input        = data.aws_availability_zones.available.names
  result_count = 3  # Take 3 from the shuffled list

  # Stable across applies for the same seed value
  keepers = {
    cluster = var.cluster_name
  }
}

# Use shuffled AZs for subnet creation
resource "aws_subnet" "private" {
  count = 3

  vpc_id            = aws_vpc.main.id
  availability_zone = random_shuffle.az_shuffle.result[count.index]
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index)

  tags = {
    Name = "private-subnet-${count.index + 1}"
    AZ   = random_shuffle.az_shuffle.result[count.index]
  }
}
```

## Step 2: Distribute EC2 Instances Across AZs

```hcl
# Assign EC2 instances to different AZs
resource "random_shuffle" "instance_azs" {
  input        = data.aws_availability_zones.available.names
  result_count = length(data.aws_availability_zones.available.names)

  keepers = { deployment = var.deployment_id }
}

# Each instance goes to a different AZ via round-robin on shuffled list
resource "aws_instance" "app" {
  count = var.instance_count

  ami               = data.aws_ami.app.id
  instance_type     = "m5.large"
  availability_zone = random_shuffle.instance_azs.result[count.index % length(random_shuffle.instance_azs.result)]

  subnet_id = aws_subnet.private[count.index % length(aws_subnet.private)].id
}
```

## Step 3: Random Primary Region Selection

```hcl
# Randomly assign which region gets primary traffic
locals {
  regions = ["us-east-1", "eu-west-1", "ap-southeast-1"]
}

resource "random_shuffle" "region_order" {
  input        = local.regions
  result_count = length(local.regions)

  keepers = {
    rotation_month = formatdate("YYYY-MM", timestamp())  # Rotate monthly
  }
}

# First region in shuffled list becomes primary
locals {
  primary_region   = random_shuffle.region_order.result[0]
  secondary_region = random_shuffle.region_order.result[1]
  tertiary_region  = random_shuffle.region_order.result[2]
}
```

## Step 4: Shuffle Items for Testing

```hcl
# Randomly select a subset of users for canary deployment
variable "all_user_groups" {
  default = ["team-alpha", "team-beta", "team-gamma", "team-delta", "team-epsilon"]
}

resource "random_shuffle" "canary_groups" {
  input        = var.all_user_groups
  result_count = 2  # Roll out to 2 random teams first

  keepers = {
    deployment_version = var.app_version
  }
}

output "canary_release_groups" {
  value       = random_shuffle.canary_groups.result
  description = "Groups selected for canary deployment"
}
```

## Summary

`random_shuffle` in OpenTofu provides randomized but stable list ordering, controlled by the `keepers` map. Using `keepers = { cluster = var.cluster_name }` ensures the same cluster always gets the same AZ ordering across plan/apply cycles, preventing resources from moving to different AZs unexpectedly. The `result_count` parameter limits how many items to take from the shuffled list, making it useful for selecting a subset of AZs or testing groups.
