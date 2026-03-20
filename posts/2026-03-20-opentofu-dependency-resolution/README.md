# How to Explain OpenTofu Dependency Resolution

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Dependencies, Graph, Concepts, Infrastructure as Code

Description: Understand how OpenTofu resolves dependencies between resources using a directed acyclic graph, and how to manage explicit and implicit dependencies.

## Introduction

OpenTofu builds a dependency graph before applying any changes. This graph determines the order in which resources are created, updated, and destroyed. Understanding how OpenTofu discovers dependencies — and how to declare them explicitly when needed — is fundamental to writing correct configurations.

## Implicit Dependencies via References

When one resource references another's attributes, OpenTofu automatically creates a dependency.

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  # This reference creates an implicit dependency:
  # aws_subnet.public → depends on → aws_vpc.main
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_internet_gateway" "main" {
  # Implicit dependency on VPC
  vpc_id = aws_vpc.main.id
}
```

## Explicit Dependencies with depends_on

Use `depends_on` when the dependency isn't expressed through attribute references.

```hcl
resource "aws_iam_role_policy" "lambda_s3" {
  name   = "lambda-s3-access"
  role   = aws_iam_role.lambda.id
  policy = data.aws_iam_policy_document.lambda_s3.json
}

resource "aws_lambda_function" "processor" {
  function_name = "data-processor"
  role          = aws_iam_role.lambda.arn

  # The Lambda needs the policy attached before it can be created
  # This dependency isn't visible through attribute references
  depends_on = [aws_iam_role_policy.lambda_s3]
}
```

## Visualizing the Dependency Graph

Use `tofu graph` to visualize dependencies.

```bash
# Generate dependency graph in DOT format
tofu graph > graph.dot

# Convert to PNG (requires graphviz)
dot -Tpng graph.dot -o graph.png

# Or view online: paste graph.dot content at dreampuf.github.io/GraphvizOnline
```

## Parallel Resource Creation

Resources without dependencies are created in parallel.

```
Dependency graph:
aws_vpc.main
├── aws_subnet.public
│   └── aws_instance.web
└── aws_subnet.private
    └── aws_db_instance.main

Execution order:
Step 1: aws_vpc.main (no dependencies)
Step 2: aws_subnet.public AND aws_subnet.private (parallel, both depend only on VPC)
Step 3: aws_instance.web AND aws_db_instance.main (parallel, each depends on their subnet)
```

## Controlling Parallelism

Adjust parallel operations when hitting API rate limits.

```bash
# Default: 10 parallel operations
tofu apply

# Reduce parallelism to avoid rate limiting
tofu apply -parallelism=3

# Increase for faster applies with high API limits
tofu apply -parallelism=20
```

## Destroy Order is Reversed

OpenTofu destroys resources in the reverse order of creation.

```
Create order:  VPC → Subnet → Security Group → Instance
Destroy order: Instance → Security Group → Subnet → VPC

This ensures dependent resources are removed before their dependencies.
```

## Dependency Cycles Are Errors

Circular dependencies prevent OpenTofu from determining execution order.

```hcl
# This creates a circular dependency — OpenTofu will error
resource "aws_security_group" "a" {
  ingress {
    from_port       = 443
    security_groups = [aws_security_group.b.id]  # depends on b
  }
}

resource "aws_security_group" "b" {
  ingress {
    from_port       = 443
    security_groups = [aws_security_group.a.id]  # depends on a → cycle!
  }
}

# Solution: use aws_security_group_rule resources to break the cycle
resource "aws_security_group" "a" {}
resource "aws_security_group" "b" {}

resource "aws_security_group_rule" "a_from_b" {
  type                     = "ingress"
  security_group_id        = aws_security_group.a.id
  source_security_group_id = aws_security_group.b.id
}

resource "aws_security_group_rule" "b_from_a" {
  type                     = "ingress"
  security_group_id        = aws_security_group.b.id
  source_security_group_id = aws_security_group.a.id
}
```

## Summary

OpenTofu builds a directed acyclic graph (DAG) of resource dependencies, where edges come from attribute references (implicit) and `depends_on` declarations (explicit). Resources with no mutual dependencies are created in parallel, maximizing speed. Circular dependencies are errors that must be resolved by restructuring the configuration. Use `tofu graph` to visualize dependencies when troubleshooting complex configurations.
