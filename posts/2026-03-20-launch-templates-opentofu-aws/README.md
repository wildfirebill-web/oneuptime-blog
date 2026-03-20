# How to Use Launch Templates with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Launch Templates, EC2, Infrastructure as Code

Description: Learn how to define and manage AWS EC2 Launch Templates using OpenTofu to standardize instance configurations and enable consistent auto-scaling deployments.

## Introduction

AWS Launch Templates allow you to define reusable EC2 instance configurations, including AMI IDs, instance types, security groups, and user data scripts. Using OpenTofu to manage Launch Templates ensures your infrastructure is version-controlled, repeatable, and easy to update.

## Prerequisites

- OpenTofu installed (v1.6+)
- AWS CLI configured with appropriate credentials
- An existing VPC and subnets in your AWS account

## What Are Launch Templates?

Launch Templates replace the older Launch Configurations and offer more flexibility. They support versioning, making it easy to roll back configurations, and can be used with Auto Scaling Groups, EC2 Fleet, and Spot Instances.

## Defining a Launch Template in OpenTofu

### Provider Configuration

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

### Variables

```hcl
variable "aws_region" {
  default = "us-east-1"
}

variable "ami_id" {
  description = "AMI ID for EC2 instances"
  type        = string
}

variable "instance_type" {
  default = "t3.medium"
}
```

### Launch Template Resource

```hcl
resource "aws_launch_template" "app_server" {
  name_prefix   = "app-server-"
  image_id      = var.ami_id
  instance_type = var.instance_type

  vpc_security_group_ids = [aws_security_group.app.id]

  iam_instance_profile {
    name = aws_iam_instance_profile.app_profile.name
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 30
      volume_type           = "gp3"
      delete_on_termination = true
      encrypted             = true
    }
  }

  user_data = base64encode(<<-EOF
    #!/bin/bash
    yum update -y
    yum install -y amazon-cloudwatch-agent
    /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
      -a start
  EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "app-server"
      Environment = "production"
    }
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

### Auto Scaling Group Using the Launch Template

```hcl
resource "aws_autoscaling_group" "app" {
  desired_capacity = 2
  max_size         = 5
  min_size         = 1
  vpc_zone_identifier = var.subnet_ids

  launch_template {
    id      = aws_launch_template.app_server.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "app-asg-instance"
    propagate_at_launch = true
  }
}
```

## Versioning Launch Templates

OpenTofu manages Launch Template versions automatically. Each `apply` that changes the template creates a new version. Reference a specific version:

```hcl
launch_template {
  id      = aws_launch_template.app_server.id
  version = aws_launch_template.app_server.latest_version
}
```

## Deploying

```bash
tofu init
tofu plan -var="ami_id=ami-0abcdef1234567890"
tofu apply -var="ami_id=ami-0abcdef1234567890"
```

## Best Practices

- Use `name_prefix` instead of `name` to allow clean replacements.
- Always encrypt EBS volumes by default.
- Store sensitive user data in AWS Secrets Manager and reference it at runtime.
- Pin Launch Template versions in production ASGs to prevent unexpected updates.
- Use `lifecycle { create_before_destroy = true }` to avoid downtime during updates.

## Conclusion

OpenTofu makes managing AWS Launch Templates straightforward with clear resource definitions and built-in versioning. By combining Launch Templates with Auto Scaling Groups, you can build resilient, consistently configured workloads that scale seamlessly.
