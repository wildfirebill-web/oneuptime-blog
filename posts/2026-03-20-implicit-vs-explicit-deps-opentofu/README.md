# How to Understand Implicit vs Explicit Dependencies in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Dependencies, depends_on, Infrastructure as Code, HCL

Description: Understand the difference between implicit and explicit dependencies in OpenTofu and when to use each to control resource creation order.

OpenTofu automatically infers most resource dependencies by analyzing attribute references in your configuration. However, some ordering requirements cannot be expressed through references alone - for those, you use `depends_on`. Understanding both types keeps your configuration clean and avoids hidden ordering bugs.

## Implicit Dependencies

An implicit dependency is created automatically when one resource references an attribute of another. OpenTofu detects these attribute references and creates a dependency edge in its internal graph.

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  # Referencing aws_vpc.main.id creates an implicit dependency.
  # OpenTofu will create the VPC before the subnet automatically.
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
```

In the graph:
```text
aws_subnet.public -> aws_vpc.main
```

No extra configuration is needed. This is the preferred and most common form of dependency.

## Explicit Dependencies with depends_on

An explicit dependency is declared with the `depends_on` meta-argument. Use it when a resource relies on the *side effects* of another resource rather than its attributes - for example, waiting for an IAM policy to propagate before a Lambda function uses it.

```hcl
resource "aws_iam_role_policy_attachment" "lambda_exec" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_lambda_function" "processor" {
  function_name = "data-processor"
  role          = aws_iam_role.lambda.arn

  # The function doesn't reference the attachment directly,
  # but it must exist before the function can execute with the right permissions.
  depends_on = [aws_iam_role_policy_attachment.lambda_exec]
}
```

## Module-Level depends_on

`depends_on` also works at the module level to ensure all resources in a module are created before resources in another module:

```hcl
module "networking" {
  source = "./modules/networking"
}

module "compute" {
  source = "./modules/compute"

  # Ensure all networking resources exist before creating any compute resources.
  # Use this only when implicit dependencies inside the modules are not sufficient.
  depends_on = [module.networking]
}
```

## When to Use Each

| Scenario | Use |
|---|---|
| Resource A reads an attribute from resource B | Implicit (attribute reference) |
| Resource A needs B's side effects (IAM propagation, DNS TTL) | Explicit (`depends_on`) |
| Module A must finish before module B starts | Explicit (`depends_on` on module) |
| Data source needs a resource to exist first | Explicit (`depends_on`) |

## Avoiding Unnecessary depends_on

Overusing `depends_on` reduces parallelism and can cause cascading rebuilds. For example:

```hcl
# BAD: unnecessary depends_on that prevents parallel creation

resource "aws_s3_bucket" "logs" { bucket = "my-logs" }

resource "aws_s3_bucket" "data" {
  bucket = "my-data"
  # These two buckets have no relationship - remove the depends_on
  depends_on = [aws_s3_bucket.logs]
}
```

The two buckets are independent. Adding `depends_on` forces them to be created sequentially when they could be created in parallel.

## Viewing Dependencies in the Graph

```bash
# Generate the plan graph to see both implicit and explicit dependencies
tofu graph -type=plan | dot -Tsvg -o deps-graph.svg
```

Both implicit and explicit dependencies appear as edges in the graph output.

## Conclusion

Prefer implicit dependencies expressed through attribute references - they are self-documenting and automatically maintained. Reserve `depends_on` for cases where resource creation order cannot be inferred from attribute references alone, such as IAM propagation delays or cross-module side effects.
