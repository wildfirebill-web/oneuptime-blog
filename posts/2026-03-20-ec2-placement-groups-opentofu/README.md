# How to Set Up EC2 Placement Groups with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, EC2, Placement Groups, High Performance Computing, Infrastructure as Code, Networking

Description: Learn how to create EC2 placement groups for cluster, spread, and partition strategies using OpenTofu to optimize instance placement for performance and availability.

## Introduction

EC2 placement groups control how instances are physically placed in AWS infrastructure. Cluster placement groups pack instances close together for low-latency networking. Spread groups place instances on distinct hardware for high availability. Partition groups divide instances across logical partitions for large distributed systems.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with EC2 permissions

## Step 1: Create a Cluster Placement Group

```hcl
# Cluster placement group packs instances into a single AZ
# for maximum network throughput and minimal latency
# Best for HPC, big data, and ML training workloads
resource "aws_placement_group" "cluster" {
  name     = "hpc-cluster"
  strategy = "cluster"

  tags = {
    Name    = "hpc-cluster-pg"
    UseCase = "HighPerformanceComputing"
  }
}

# Launch HPC instances into the cluster placement group
resource "aws_instance" "hpc" {
  count             = 4
  ami               = data.aws_ami.amazon_linux.id
  instance_type     = "c5n.18xlarge"  # Network-optimized instance
  subnet_id         = var.subnet_id
  placement_group   = aws_placement_group.cluster.id

  tags = {
    Name = "hpc-node-${count.index + 1}"
  }
}
```

## Step 2: Create a Spread Placement Group

```hcl
# Spread placement group places each instance on distinct hardware
# Maximum of 7 instances per AZ per spread group
# Best for critical applications requiring high availability
resource "aws_placement_group" "spread" {
  name     = "critical-app-spread"
  strategy = "spread"

  tags = {
    Name    = "critical-spread-pg"
    UseCase = "HighAvailability"
  }
}

resource "aws_instance" "critical" {
  count           = 3  # Max 7 per AZ
  ami             = data.aws_ami.amazon_linux.id
  instance_type   = "m5.large"
  subnet_id       = var.subnet_id
  placement_group = aws_placement_group.spread.id

  tags = {
    Name = "critical-instance-${count.index + 1}"
  }
}
```

## Step 3: Create a Partition Placement Group

```hcl
# Partition placement group divides instances across logical partitions
# Each partition uses distinct underlying hardware racks
# Best for Hadoop, Cassandra, Kafka distributed workloads
resource "aws_placement_group" "partition" {
  name            = "kafka-partition"
  strategy        = "partition"
  partition_count = 3  # Number of partitions (max 7)

  tags = {
    Name    = "kafka-partition-pg"
    UseCase = "DistributedSystems"
  }
}

# Launch Kafka brokers, specifying which partition each goes into
resource "aws_instance" "kafka" {
  count           = 3
  ami             = data.aws_ami.amazon_linux.id
  instance_type   = "m5.xlarge"
  subnet_id       = var.subnet_id
  placement_group = aws_placement_group.partition.id

  # Each Kafka broker in a separate partition for fault tolerance
  # partition_number is set in the instance resource (1-indexed)

  tags = {
    Name      = "kafka-broker-${count.index + 1}"
    Partition = count.index + 1
  }
}
```

## Step 4: Use Placement Groups with Launch Templates

```hcl
# Reference a placement group in a launch template
# for use with Auto Scaling Groups
resource "aws_launch_template" "hpc" {
  name          = "hpc-launch-template"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = "c5n.9xlarge"

  placement {
    group_name = aws_placement_group.cluster.name
  }
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

EC2 placement groups give you control over instance placement to meet specific performance and availability requirements. Use cluster groups for HPC workloads needing sub-millisecond latency, spread groups for mission-critical single-instance applications, and partition groups for large-scale distributed systems that need rack-level fault tolerance. Note that cluster groups require instances to be in the same AZ and instance type.
