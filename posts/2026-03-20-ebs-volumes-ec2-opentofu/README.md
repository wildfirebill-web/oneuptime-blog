# How to Attach EBS Volumes to EC2 Instances with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, EC2, EBS, Block Storage, Infrastructure as Code, Storage

Description: Learn how to create EBS volumes and attach them to EC2 instances using OpenTofu, including configuration of volume types, IOPS, and throughput for different workload requirements.

## Introduction

Amazon EBS (Elastic Block Store) provides persistent block storage for EC2 instances. Unlike instance store volumes, EBS volumes persist independently from the instance lifecycle. This guide covers creating and attaching data volumes with appropriate configurations for various workload types.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with EC2 and EBS permissions
- An existing EC2 instance or subnet

## Step 1: Launch an EC2 Instance

```hcl
resource "aws_instance" "database" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "m5.xlarge"
  subnet_id     = var.subnet_id

  # Root volume - OS disk
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 50
    iops                  = 3000
    throughput            = 125
    encrypted             = true
    delete_on_termination = true
  }

  tags = { Name = "database-instance" }
}
```

## Step 2: Create Additional EBS Volumes

```hcl
# High-performance io2 volume for database data files
resource "aws_ebs_volume" "database_data" {
  availability_zone = aws_instance.database.availability_zone
  type              = "io2"
  size              = 500   # 500 GiB
  iops              = 16000 # Up to 64,000 IOPS for io2
  encrypted         = true
  kms_key_id        = var.kms_key_arn

  tags = {
    Name     = "database-data-volume"
    Instance = aws_instance.database.id
    Purpose  = "DatabaseData"
  }
}

# gp3 volume for database logs - cost-effective with good throughput
resource "aws_ebs_volume" "database_logs" {
  availability_zone = aws_instance.database.availability_zone
  type              = "gp3"
  size              = 200
  iops              = 3000
  throughput        = 250  # MiB/s, up to 1000 for gp3
  encrypted         = true

  tags = {
    Name    = "database-log-volume"
    Purpose = "DatabaseLogs"
  }
}

# st1 volume for backups - optimized for sequential throughput
resource "aws_ebs_volume" "database_backup" {
  availability_zone = aws_instance.database.availability_zone
  type              = "st1"  # Throughput-optimized HDD
  size              = 2000   # 2 TiB for backup storage

  tags = {
    Name    = "database-backup-volume"
    Purpose = "Backups"
  }
}
```

## Step 3: Attach Volumes to the Instance

```hcl
# Attach the data volume as /dev/sdb
resource "aws_volume_attachment" "data" {
  device_name = "/dev/sdb"
  volume_id   = aws_ebs_volume.database_data.id
  instance_id = aws_instance.database.id

  # Stop the instance to detach (set to false for hot-detach capable instances)
  stop_instance_before_detaching = false
}

resource "aws_volume_attachment" "logs" {
  device_name = "/dev/sdc"
  volume_id   = aws_ebs_volume.database_logs.id
  instance_id = aws_instance.database.id
}

resource "aws_volume_attachment" "backup" {
  device_name = "/dev/sdd"
  volume_id   = aws_ebs_volume.database_backup.id
  instance_id = aws_instance.database.id
}
```

## Step 4: Format and Mount via User Data

```hcl
# User data script to format and mount the attached volumes
locals {
  user_data = <<-EOF
    #!/bin/bash
    # Wait for volumes to be available
    sleep 10

    # Format and mount data volume
    mkfs -t xfs /dev/sdb
    mkdir -p /data
    mount /dev/sdb /data
    echo '/dev/sdb /data xfs defaults 0 0' >> /etc/fstab

    # Format and mount log volume
    mkfs -t xfs /dev/sdc
    mkdir -p /logs
    mount /dev/sdc /logs
    echo '/dev/sdc /logs xfs defaults 0 0' >> /etc/fstab
  EOF
}
```

## Step 5: Outputs

```hcl
output "data_volume_id" {
  value = aws_ebs_volume.database_data.id
}

output "data_volume_iops" {
  value = aws_ebs_volume.database_data.iops
}
```

## Step 6: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

You have attached multiple EBS volumes with appropriate types for different I/O patterns: io2 for high-IOPS database data, gp3 for balanced log storage, and st1 for high-throughput sequential backup workloads. Always encrypt EBS volumes containing sensitive data and use volume tags to track costs and ownership across your infrastructure.
