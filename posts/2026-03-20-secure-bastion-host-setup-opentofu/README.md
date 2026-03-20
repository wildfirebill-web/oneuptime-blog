# How to Build a Secure Bastion Host Setup with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Bastion Host, Security, SSH, AWS, Infrastructure as Code

Description: Learn how to build a secure bastion host setup on AWS with OpenTofu, including SSH key management, security groups, EC2 Instance Connect, and audit logging.

## Introduction

A bastion host provides secure SSH access to private network resources. Modern bastion designs minimize attack surface by using EC2 Instance Connect (which eliminates permanent SSH keys), strict security group rules, CloudWatch audit logging, and auto-shutdown policies. This guide builds a production-grade bastion with these features.

## Security Group for Bastion

```hcl
resource "aws_security_group" "bastion" {
  name        = "bastion-sg"
  description = "Security group for bastion host - restrict to known IPs"
  vpc_id      = module.vpc.vpc_id

  # SSH access only from approved IP ranges
  ingress {
    description = "SSH from office IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.allowed_ssh_cidrs  # ["203.0.113.0/24"]
  }

  # No direct internet egress - route through NAT
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["10.0.0.0/8"]  # only internal traffic
  }

  tags = {
    Name        = "bastion-sg"
    Environment = var.environment
  }
}
```

## EC2 Instance with Instance Connect

```hcl
resource "aws_instance" "bastion" {
  ami                    = data.aws_ami.amazon_linux_2023.id
  instance_type          = "t3.micro"
  subnet_id              = module.vpc.public_subnets[0]
  vpc_security_group_ids = [aws_security_group.bastion.id]
  iam_instance_profile   = aws_iam_instance_profile.bastion.name

  # EC2 Instance Connect eliminates need for long-lived SSH keys
  # Users connect via: aws ec2-instance-connect send-ssh-public-key + ssh
  key_name = null  # no persistent key pair

  metadata_options {
    http_tokens   = "required"  # IMDSv2 required
    http_endpoint = "enabled"
  }

  root_block_device {
    volume_type = "gp3"
    volume_size = 20
    encrypted   = true
  }

  user_data = base64encode(<<-EOF
    #!/bin/bash
    # Install EC2 Instance Connect
    yum install -y ec2-instance-connect

    # Configure SSH to log all sessions to CloudWatch
    yum install -y awslogs

    # Configure bastion audit logging
    cat >> /etc/ssh/sshd_config << 'SSHD'
    LogLevel VERBOSE
    PrintLastLog yes
    SSHD

    systemctl restart sshd
  EOF
  )

  tags = {
    Name        = "bastion-${var.environment}"
    Environment = var.environment
    AutoStop    = "true"  # tag for auto-shutdown Lambda
  }
}
```

## IAM Role for Bastion

```hcl
resource "aws_iam_role" "bastion" {
  name = "bastion-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "bastion_ssm" {
  role       = aws_iam_role.bastion.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "bastion" {
  name = "bastion-profile"
  role = aws_iam_role.bastion.name
}
```

## Security Group for Private Resources

```hcl
# Allow SSH from bastion to private instances
resource "aws_security_group" "private_instances" {
  name        = "private-instances-sg"
  description = "Allow SSH from bastion only"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description     = "SSH from bastion"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }
}
```

## Alternative: AWS Systems Manager Session Manager

No bastion needed — use SSM for zero-SSH access.

```hcl
# With SSM Session Manager, no bastion host is needed at all
# Instances need SSM agent and role:

resource "aws_iam_role_policy_attachment" "app_ssm" {
  role       = aws_iam_role.app_instance.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# Connect via:
# aws ssm start-session --target i-0abc12345def67890
# No SSH, no bastion, no open ports, all sessions logged
```

## Auto-Stop the Bastion When Idle

```hcl
resource "aws_cloudwatch_metric_alarm" "bastion_idle" {
  alarm_name          = "bastion-idle"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 30  # 30 minutes of no CPU usage
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 60
  statistic           = "Average"
  threshold           = 1.0
  alarm_description   = "Stop bastion if idle for 30 minutes"
  alarm_actions       = [aws_sns_topic.bastion_stop.arn]

  dimensions = {
    InstanceId = aws_instance.bastion.id
  }
}
```

## Summary

A secure bastion host uses EC2 Instance Connect for temporary SSH keys (no persistent key pairs), IMDSv2 enforcement, encrypted EBS volumes, restrictive security groups allowing SSH only from known CIDRs, and CloudWatch logging for audit trails. Consider replacing the traditional bastion entirely with AWS Systems Manager Session Manager — it requires no open SSH ports, logs all session activity, and integrates with IAM for access control. The SSM approach eliminates the bastion host attack surface entirely.
