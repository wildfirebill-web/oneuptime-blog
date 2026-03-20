# How to Configure EC2 Hibernation with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, EC2, Hibernation, Cost Optimization, Infrastructure as Code, Storage

Description: Learn how to enable EC2 instance hibernation using OpenTofu so instances can save their in-memory state to EBS and resume quickly without a full boot process.

## Introduction

EC2 hibernation saves the instance RAM contents to the root EBS volume, allowing instances to resume where they left off. This is useful for long-running processes, development environments, and workloads that benefit from pre-warmed caches while avoiding continuous compute costs.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with EC2 permissions
- Root EBS volume must have enough free space to store RAM contents
- Instance type must support hibernation (most general purpose, memory optimized, and compute optimized types)

## Step 1: Enable Hibernation on an EC2 Instance

```hcl
resource "aws_instance" "hibernatable" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.medium"  # Must support hibernation
  subnet_id     = var.subnet_id

  # Hibernation requires an encrypted root volume
  root_block_device {
    volume_type           = "gp3"
    # Volume size must exceed RAM size to store hibernation image
    # t3.medium has 4 GiB RAM, so 30 GiB provides ample space
    volume_size           = 30
    encrypted             = true  # Required for hibernation
    delete_on_termination = true
  }

  # Enable hibernation
  hibernation = true

  # IMDSv2 enforced for security
  metadata_options {
    http_tokens = "required"
  }

  tags = {
    Name          = "hibernatable-instance"
    Hibernation   = "enabled"
    Environment   = var.environment
  }
}
```

## Step 2: Use Hibernation with a Launch Template

```hcl
# Launch template with hibernation for Auto Scaling Groups

# Note: ASGs must be configured to stop instances for scale-in
# hibernation is for individual instance suspend/resume
resource "aws_launch_template" "hibernate" {
  name          = "hibernate-launch-template"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = "m5.large"

  # Hibernation requires encrypted root volume
  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 40
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }

  hibernation_options {
    configured = true
  }

  metadata_options {
    http_tokens = "required"
  }

  tags = { Name = "hibernate-template" }
}
```

## Step 3: Hibernate and Resume via AWS CLI

```bash
# Hibernate a running instance
aws ec2 stop-instances \
  --instance-ids i-0123456789abcdef0 \
  --hibernate \
  --region us-east-1

# Check hibernation state
aws ec2 describe-instances \
  --instance-ids i-0123456789abcdef0 \
  --query 'Reservations[0].Instances[0].StateReason.Message'

# Start the instance (resumes from hibernate)
aws ec2 start-instances \
  --instance-ids i-0123456789abcdef0 \
  --region us-east-1
```

## Step 4: Verify Hibernation Prerequisites

```hcl
# Ensure root volume size is large enough for the instance RAM
locals {
  # Map of instance types to RAM sizes (GiB)
  instance_ram = {
    "t3.medium"  = 4
    "t3.large"   = 8
    "m5.large"   = 8
    "m5.xlarge"  = 16
    "m5.2xlarge" = 32
  }

  required_root_size = local.instance_ram[var.instance_type] + 10
}

output "minimum_root_volume_size" {
  description = "Minimum EBS root volume size for hibernation"
  value       = "${local.required_root_size} GiB"
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

EC2 hibernation is a powerful feature for workloads with long initialization times or pre-warmed caches. The instance resumes within seconds rather than minutes because only the memory state must be restored. Remember that instances cannot be hibernated if they have been running for more than 60 days, and hibernation is not supported for bare metal instances or instances with more than 150 GiB of RAM.
