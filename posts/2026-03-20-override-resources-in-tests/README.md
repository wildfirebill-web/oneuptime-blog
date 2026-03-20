# How to Override Resources in OpenTofu Tests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Override Resources, Unit Testing, Infrastructure as Code

Description: Learn how to use override_resource in OpenTofu tests to control resource attribute values during tests, enabling assertions on dependent resources without full provisioning.

## Introduction

OpenTofu's `override_resource` directive allows you to replace a resource's computed attributes with controlled values during a test run. This is useful when testing resources that depend on values only known after other resources are created, such as ARNs, IDs, and dynamically assigned IPs.

## Why Override Resources

Override resources in tests when:
- A resource's computed attributes are needed by other resources
- You want to test how dependent resources use a specific value
- You're writing apply-mode tests without real cloud credentials
- You need deterministic values for assertion checks

## Basic Resource Override

```hcl
# main.tf

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_subnet.private.cidr_block]
  }
}
```

```hcl
# tests/resources.tftest.hcl

mock_provider "aws" {
  mock_resource "aws_vpc" {
    defaults = { id = "vpc-default" }
  }
  mock_resource "aws_subnet" {
    defaults = { id = "subnet-default", cidr_block = "10.0.99.0/24" }
  }
  mock_resource "aws_security_group" {
    defaults = { id = "sg-default" }
  }
}

run "sg_ingress_uses_subnet_cidr" {
  command = apply

  override_resource {
    target = aws_subnet.private
    values = {
      id         = "subnet-test123"
      cidr_block = "10.0.1.0/24"
      vpc_id     = "vpc-mock"
    }
  }

  assert {
    condition = aws_security_group.app.ingress[0].cidr_blocks == toset(["10.0.1.0/24"])
    error_message = "Security group ingress should use subnet CIDR"
  }
}
```

## Overriding Resource ARNs

Test IAM policy documents that reference resource ARNs:

```hcl
# main.tf

resource "aws_s3_bucket" "data" {
  bucket = "mycompany-data"
}

data "aws_iam_policy_document" "data_access" {
  statement {
    effect    = "Allow"
    actions   = ["s3:GetObject", "s3:PutObject"]
    resources = [
      aws_s3_bucket.data.arn,
      "${aws_s3_bucket.data.arn}/*"
    ]
  }
}

resource "aws_iam_policy" "data_access" {
  name   = "data-access"
  policy = data.aws_iam_policy_document.data_access.json
}
```

```hcl
run "policy_uses_bucket_arn" {
  command = plan

  override_resource {
    target = aws_s3_bucket.data
    values = {
      id  = "mycompany-data"
      arn = "arn:aws:s3:::mycompany-data"
    }
  }

  assert {
    condition     = contains(
      jsondecode(aws_iam_policy.data_access.policy).Statement[0].Resource,
      "arn:aws:s3:::mycompany-data"
    )
    error_message = "Policy should reference the bucket ARN"
  }
}
```

## Overriding After Create

In apply-mode tests, override the post-creation values:

```hcl
# main.tf

resource "aws_lb" "app" {
  name               = "app-alb"
  internal           = false
  load_balancer_type = "application"
}

resource "aws_route53_record" "app" {
  zone_id = var.hosted_zone_id
  name    = "app.example.com"
  type    = "A"

  alias {
    name                   = aws_lb.app.dns_name
    zone_id                = aws_lb.app.zone_id
    evaluate_target_health = true
  }
}
```

```hcl
mock_provider "aws" {
  mock_resource "aws_lb" {
    defaults = {
      dns_name = "app-alb-default.us-east-1.elb.amazonaws.com"
      zone_id  = "Z35SXDOTRQ7X7K"
    }
  }
  mock_resource "aws_route53_record" {
    defaults = { id = "ZONEID_app.example.com_A" }
  }
}

run "dns_alias_points_to_alb" {
  command = apply

  override_resource {
    target = aws_lb.app
    values = {
      dns_name = "test-alb.us-east-1.elb.amazonaws.com"
      zone_id  = "Z35SXDOTRQ7X7K"
    }
  }

  assert {
    condition     = aws_route53_record.app.alias[0].name == "test-alb.us-east-1.elb.amazonaws.com"
    error_message = "DNS record should alias the ALB DNS name"
  }
}
```

## Combining override_resource with expect_failures

Test that a resource rejects a bad value from a dependency:

```hcl
run "rejects_invalid_vpc" {
  command = plan

  override_resource {
    target = aws_vpc.main
    values = {
      id         = ""  # Invalid empty ID
      cidr_block = "10.0.0.0/16"
    }
  }

  # Expect the subnet resource to fail due to empty vpc_id
  expect_failures = [aws_subnet.private]
}
```

## Best Practices

- Use `override_resource` in combination with `mock_provider` for fully offline tests.
- Set override values that represent realistic production values.
- Prefer explicit references in configurations (e.g., `aws_vpc.main.id`) over hardcoded values so that overrides propagate correctly.
- Use `apply` command in runs where you need computed attributes to be available for assertions.
- Document why specific override values were chosen in test comments.

## Conclusion

`override_resource` gives you precise control over resource attribute values in OpenTofu tests. Combined with mock providers, it enables a fully offline test suite that validates how your configuration uses resource attributes — without any cloud API calls or infrastructure costs.
