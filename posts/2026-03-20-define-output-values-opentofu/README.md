# How to Define Output Values in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Outputs, HCL, Infrastructure as Code, DevOps

Description: A guide to defining and using output values in OpenTofu to expose infrastructure information after deployment.

## Introduction

Output values in OpenTofu expose resource attributes and computed values after a `tofu apply`. They are used to display useful information, pass data between modules, and integrate with other systems. This guide covers defining outputs comprehensively.

## Basic Output Definition

```hcl
# outputs.tf

# Simple output

output "vpc_id" {
  value = aws_vpc.main.id
}

# Output with description
output "instance_public_ip" {
  description = "The public IP address of the web server"
  value       = aws_instance.web.public_ip
}

# Sensitive output
output "database_password" {
  description = "The generated database password"
  value       = aws_db_instance.main.password
  sensitive   = true
}
```

## Output Types

```hcl
# String output
output "bucket_name" {
  description = "Name of the S3 bucket"
  value       = aws_s3_bucket.main.id
}

# Number output
output "instance_count" {
  description = "Number of running instances"
  value       = length(aws_instance.web)
}

# Boolean output
output "is_multi_az" {
  description = "Whether the database is Multi-AZ"
  value       = aws_db_instance.main.multi_az
}

# List output
output "subnet_ids" {
  description = "IDs of all subnets"
  value       = aws_subnet.public[*].id
}

# Map output
output "instance_ips" {
  description = "Map of instance name to IP address"
  value       = { for k, v in aws_instance.web : k => v.public_ip }
}

# Object output
output "database_info" {
  description = "Database connection information"
  value = {
    endpoint = aws_db_instance.main.endpoint
    port     = aws_db_instance.main.port
    name     = aws_db_instance.main.db_name
  }
}
```

## Outputs from count Resources

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# List of IPs from count resources
output "web_ips" {
  description = "Public IPs of all web instances"
  value       = aws_instance.web[*].public_ip
  # Returns: ["1.2.3.4", "5.6.7.8", "9.10.11.12"]
}

# Specific instance
output "first_web_ip" {
  description = "Public IP of the first web instance"
  value       = aws_instance.web[0].public_ip
}
```

## Outputs from for_each Resources

```hcl
resource "aws_instance" "web" {
  for_each = toset(["web-1", "web-2", "web-3"])

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# Map of instance name to IP
output "web_instance_ips" {
  description = "Map of instance name to public IP"
  value       = { for k, v in aws_instance.web : k => v.public_ip }
  # Returns: {"web-1": "1.2.3.4", "web-2": "5.6.7.8", "web-3": "9.10.11.12"}
}
```

## Computed Outputs

```hcl
output "alb_dns_name" {
  description = "DNS name of the Application Load Balancer"
  value       = aws_lb.main.dns_name
}

output "connection_string" {
  description = "Database connection string"
  value       = "postgresql://${aws_db_instance.main.username}@${aws_db_instance.main.endpoint}/${aws_db_instance.main.db_name}"
  sensitive   = true  # Contains endpoint info
}

output "kubectl_config_command" {
  description = "Command to configure kubectl"
  value       = "aws eks update-kubeconfig --name ${aws_eks_cluster.main.name} --region ${var.aws_region}"
}
```

## Viewing Outputs

```bash
# Show all outputs
tofu output

# Show specific output
tofu output vpc_id

# Get raw value (no quotes)
tofu output -raw vpc_id

# Get all outputs as JSON
tofu output -json

# Get specific output as JSON
tofu output -json database_info | jq .endpoint
```

## Conclusion

Output values are essential for making your OpenTofu infrastructure usable - they expose the information needed to connect to, configure, or reference your created resources. Well-documented outputs with clear descriptions make modules easier to use. Sensitive outputs protect credentials from casual exposure while still making them programmatically accessible when needed.
