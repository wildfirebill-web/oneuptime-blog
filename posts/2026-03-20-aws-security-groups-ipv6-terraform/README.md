# How to Configure AWS Security Groups for IPv6 with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, Terraform, IPv6, Security Groups, Networking, Firewall

Description: A guide to writing AWS Security Group rules that cover IPv6 traffic in Terraform, including common dual-stack patterns.

AWS Security Groups control traffic at the instance level. IPv4 and IPv6 rules are specified separately - an IPv4 CIDR rule does not automatically apply to IPv6. This means dual-stack security groups must explicitly include both `cidr_blocks` (IPv4) and `ipv6_cidr_blocks` (IPv6) in each rule.

## Key Difference: IPv4 vs IPv6 Rules

In Terraform's `aws_security_group` resource:
- `cidr_blocks` - list of IPv4 CIDR ranges
- `ipv6_cidr_blocks` - list of IPv6 CIDR ranges
- Both are required for dual-stack coverage

## Step 1: Create a Dual-Stack Web Security Group

```hcl
# sg-web.tf - Security group allowing HTTP/HTTPS from both IPv4 and IPv6

resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Allow HTTP and HTTPS from internet (IPv4 and IPv6)"
  vpc_id      = aws_vpc.main.id

  # Allow HTTP from all IPv4
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP from IPv4 internet"
  }

  # Allow HTTP from all IPv6
  ingress {
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    ipv6_cidr_blocks = ["::/0"]
    description      = "HTTP from IPv6 internet"
  }

  # Allow HTTPS from all IPv4
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS from IPv4 internet"
  }

  # Allow HTTPS from all IPv6
  ingress {
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    ipv6_cidr_blocks = ["::/0"]
    description      = "HTTPS from IPv6 internet"
  }

  # Allow all outbound traffic (IPv4)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound IPv4"
  }

  # Allow all outbound traffic (IPv6)
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    ipv6_cidr_blocks = ["::/0"]
    description      = "All outbound IPv6"
  }

  tags = {
    Name = "web-sg"
  }
}
```

## Step 2: Allow ICMPv6 (Required for IPv6 Functionality)

ICMPv6 is essential for IPv6 Neighbor Discovery, Path MTU Discovery, and other core protocols. Always allow it:

```hcl
# sg-icmpv6.tf - Allow ICMPv6 (protocol number 58)
resource "aws_security_group_rule" "allow_icmpv6_ingress" {
  security_group_id = aws_security_group.web.id
  type              = "ingress"
  protocol          = "58"  # ICMPv6 protocol number
  from_port         = -1
  to_port           = -1
  ipv6_cidr_blocks  = ["::/0"]
  description       = "Allow all ICMPv6 (required for NDP and PMTUD)"
}
```

## Step 3: Restrict SSH to a Specific IPv6 Prefix

```hcl
# sg-ssh.tf - Allow SSH from a specific IPv6 management prefix
resource "aws_security_group_rule" "ssh_ipv6" {
  security_group_id = aws_security_group.web.id
  type              = "ingress"
  protocol          = "tcp"
  from_port         = 22
  to_port           = 22
  # Replace with your management IPv6 prefix
  ipv6_cidr_blocks  = ["2001:db8:management::/48"]
  description       = "SSH from management IPv6 prefix"
}
```

## Step 4: Use Variables for Reusable Rules

```hcl
# variables.tf
variable "allowed_ipv6_cidrs" {
  type        = list(string)
  description = "IPv6 CIDR blocks allowed to access the application"
  default     = ["::/0"]
}

# Use the variable in the security group rule
ingress {
  from_port        = 8080
  to_port          = 8080
  protocol         = "tcp"
  ipv6_cidr_blocks = var.allowed_ipv6_cidrs
  description      = "App port from allowed IPv6 ranges"
}
```

## Step 5: Apply and Verify

```bash
terraform apply

# Confirm IPv6 rules are present in the security group
aws ec2 describe-security-groups \
  --group-ids $(terraform output -raw web_sg_id) \
  --query 'SecurityGroups[0].IpPermissions[*].Ipv6Ranges'
```

Always pair every IPv4 rule with an equivalent IPv6 rule in dual-stack deployments - missing IPv6 rules are the most common cause of asymmetric connectivity in AWS dual-stack architectures.
