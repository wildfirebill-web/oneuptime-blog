# Understanding OpenTofu's Dependency Graph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Dependency Graph, Internals, Infrastructure as Code, DevOps

Description: Learn how OpenTofu builds and traverses the resource dependency graph - understanding implicit and explicit dependencies, cycle detection, and how the graph determines apply order.

## Introduction

OpenTofu builds a directed acyclic graph (DAG) of all resources in your configuration before applying changes. This graph determines the order of resource creation, modification, and destruction. Understanding how it works helps you write efficient configurations and debug ordering issues.

## How Dependencies Are Created

### Implicit Dependencies (Reference-Based)

When you reference an attribute from another resource, OpenTofu automatically creates a dependency edge:

```hcl
# aws_subnet depends on aws_vpc because of aws_vpc.main.id reference

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id   # Implicit dependency on aws_vpc.main
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "web" {
  subnet_id = aws_subnet.public.id   # Implicit dependency on aws_subnet.public
  ami       = "ami-0abcdef1234567890"
}
```

OpenTofu's dependency graph for this config:

```text
aws_vpc.main
    └── aws_subnet.public
            └── aws_instance.web
```

### Explicit Dependencies (depends_on)

Use `depends_on` when you have a dependency that can't be expressed through attribute references:

```hcl
# IAM policy might need to be fully active before EC2 starts
resource "aws_iam_role_policy" "ec2_policy" {
  name   = "ec2-policy"
  role   = aws_iam_role.ec2.id
  policy = data.aws_iam_policy_document.ec2.json
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.latest.id
  instance_type = "t3.medium"
  iam_instance_profile = aws_iam_instance_profile.ec2.name

  # Explicit dependency - ensure policy is attached before instance starts
  depends_on = [aws_iam_role_policy.ec2_policy]
}
```

## Apply Order: How the Graph Is Traversed

OpenTofu traverses the graph in topological order:

```text
Create order:
1. aws_vpc.main           (no dependencies)
2. aws_subnet.public      (after aws_vpc.main)
3. aws_security_group.web (after aws_vpc.main)
4. aws_instance.web       (after subnet + security group)

Parallel execution:
  Step 1: aws_vpc.main
  Step 2: aws_subnet.public AND aws_security_group.web (parallel)
  Step 3: aws_instance.web
```

The `-parallelism=N` flag controls how many resources are processed concurrently within a level.

## Destroy Order

Destroy reverses the dependency graph:

```text
Destroy order (reverse of create):
1. aws_instance.web       (destroy first)
2. aws_subnet.public      (after instance is gone)
3. aws_security_group.web (after instance is gone)
4. aws_vpc.main           (after subnet and SG are gone)
```

## Module Dependencies

Modules can also form dependency relationships:

```hcl
module "vpc" {
  source = "./modules/vpc"
}

module "eks" {
  source = "./modules/eks"

  # Implicit dependency through output reference
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids
}

module "monitoring" {
  source = "./modules/monitoring"

  # Explicit dependency on EKS being ready
  depends_on = [module.eks]

  cluster_name = module.eks.cluster_name
}
```

## Visualizing the Graph

```bash
# Generate DOT format graph
tofu graph > graph.dot

# Render to PNG (requires Graphviz)
tofu graph | dot -Tpng > graph.png

# Render to SVG
tofu graph | dot -Tsvg > graph.svg

# Show plan-specific graph (what would change)
tofu plan -out=tfplan.binary
tofu graph -plan=tfplan.binary | dot -Tpng > plan-graph.png
```

## Cycle Detection

OpenTofu detects circular dependencies and reports them clearly:

```hcl
# This would create a cycle:
resource "aws_security_group" "a" {
  ingress {
    security_groups = [aws_security_group.b.id]  # A depends on B
  }
}

resource "aws_security_group" "b" {
  ingress {
    security_groups = [aws_security_group.a.id]  # B depends on A - CYCLE!
  }
}
```

```text
Error: Cycle: aws_security_group.a, aws_security_group.b
```

Fix using `aws_security_group_rule`:

```hcl
resource "aws_security_group" "a" { name = "sg-a" }
resource "aws_security_group" "b" { name = "sg-b" }

# Break the cycle with separate rule resources
resource "aws_security_group_rule" "a_to_b" {
  type                     = "ingress"
  security_group_id        = aws_security_group.a.id
  source_security_group_id = aws_security_group.b.id
  protocol                 = "tcp"
  from_port                = 443
  to_port                  = 443
}
```

## Conclusion

OpenTofu's dependency graph is the engine that determines resource creation order, enables parallel execution, and prevents partial deployments. Implicit dependencies through attribute references are the recommended approach - they self-document which resources depend on which. Use `depends_on` only for side-effect dependencies that can't be expressed through references. Use `tofu graph | dot -Tpng` to visualize complex dependency chains when debugging ordering issues.
