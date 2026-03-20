# How to Debug Data Source Read Failures in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Troubleshooting, Data Sources, Debugging, Error, Infrastructure as Code

Description: Learn how to diagnose and fix data source read failures in OpenTofu, including filter mismatches, permission errors, and dependency ordering issues.

## Introduction

Data sources in OpenTofu read existing infrastructure information at plan/apply time. When a data source read fails, it blocks all resources that depend on its output. Common causes include incorrect filters, authentication issues, and resources that don't yet exist when the data source tries to read them.

## Common Data Source Errors

```
Error: no matching EC2 VPC found
  with data.aws_vpc.main, on main.tf line 5, in data "aws_vpc" "main":
  Your query returned no results. Please change your search criteria and try again.

Error: reading IAM Role (my-role): NoSuchEntity: The role with name my-role cannot be found.

Error: Error reading S3 Bucket (my-bucket): NoSuchBucket: The specified bucket does not exist
```

## Fix 1: Filter Not Matching Any Resources

Check that your filter values exactly match the tag or attribute on the resource:

```hcl
# WRONG — tag key is case-sensitive
data "aws_vpc" "main" {
  filter {
    name   = "tag:name"    # Wrong — tag key is "Name", not "name"
    values = ["prod-vpc"]
  }
}

# CORRECT
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["prod-vpc"]
  }
}
```

Debug the filter by querying the AWS CLI:

```bash
# Verify the tag exists with the exact case
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=prod-vpc" \
  --query "Vpcs[0].{VpcId:VpcId,Tags:Tags}"
```

## Fix 2: Data Source Resource Does Not Exist Yet

When a data source references a resource that is being created in the same apply:

```hcl
# PROBLEM — the security group doesn't exist yet when data source runs
resource "aws_security_group" "app" {
  name = "app-sg"
  # ...
}

data "aws_security_group" "app" {
  filter {
    name   = "group-name"
    values = ["app-sg"]
  }
}

# SOLUTION — reference the resource directly instead of a data source
resource "aws_instance" "web" {
  vpc_security_group_ids = [aws_security_group.app.id]  # Direct reference
}
```

## Fix 3: Handling Data Source Failures Gracefully

Use `try()` or `lookup()` to handle cases where the data source may fail:

```hcl
# Null resource as a fallback when data source might not exist
data "aws_ami" "custom" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["my-custom-ami-*"]
  }
}

# Use try() to fall back to a default AMI if the custom one doesn't exist
locals {
  ami_id = try(data.aws_ami.custom.id, data.aws_ami.amazon_linux.id)
}
```

## Fix 4: Permission Denied Reading Data Source

```bash
# Enable debug logging to see the exact error
TF_LOG=DEBUG tofu plan 2>&1 | grep -A 5 "data.aws_vpc.main"

# Check IAM permissions for the data source action
# For aws_vpc: ec2:DescribeVpcs
# For aws_s3_bucket: s3:GetBucketLocation, s3:ListBucket
# For aws_iam_role: iam:GetRole
```

## Fix 5: depends_on for Data Sources

If a data source needs to read after another resource is created:

```hcl
resource "aws_s3_bucket" "app" {
  bucket = "my-app-bucket"
}

# Wait for the bucket to be created before reading
data "aws_s3_bucket" "app" {
  bucket = aws_s3_bucket.app.bucket
  # Direct reference to the resource implies an implicit dependency
  # so the data source reads after the resource is created
}
```

For cases where the implicit dependency is not enough:

```hcl
data "aws_iam_policy" "readonly" {
  name = "ReadOnlyPolicy"

  depends_on = [aws_iam_policy.readonly]  # Explicit dependency
}
```

## Conclusion

Data source read failures are debugged by checking filter case-sensitivity against actual resource tags, replacing same-apply data source lookups with direct resource references, adding `depends_on` for ordering, and using `TF_LOG=DEBUG` to surface the exact API error and required IAM permissions.
