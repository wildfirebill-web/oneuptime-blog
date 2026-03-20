# How to Handle Eventual Consistency Issues in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Eventual Consistency, IAM, AWS, Infrastructure as Code

Description: Learn techniques for handling eventual consistency issues in OpenTofu where cloud resources take time to propagate before dependent operations can succeed.

Eventual consistency is a property of distributed cloud systems where changes take time to propagate globally. In OpenTofu, this manifests as failures where a resource is created successfully but a dependent resource cannot use it yet — the classic example is creating an IAM role and immediately attaching a Lambda function that cannot find the permissions.

## Common Eventual Consistency Scenarios

| Scenario | Typical Propagation Delay |
|---|---|
| IAM role/policy propagation | 5–30 seconds |
| Route 53 DNS record propagation | Up to 60 seconds |
| AWS Certificate Manager validation | Minutes |
| S3 bucket policy propagation | A few seconds |
| API Gateway deployment | 10–30 seconds |
| VPC endpoint propagation | 30–60 seconds |

## Strategy 1: time_sleep Resource

The `time_sleep` resource from the `hashicorp/time` provider adds an explicit pause:

```hcl
terraform {
  required_providers {
    time = {
      source  = "hashicorp/time"
      version = "~> 0.9"
    }
  }
}

resource "aws_iam_role" "lambda_exec" {
  name = "lambda-execution-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_exec" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# Wait for IAM to propagate before creating the Lambda
resource "time_sleep" "wait_for_iam" {
  depends_on      = [aws_iam_role_policy_attachment.lambda_exec]
  create_duration = "15s"
}

resource "aws_lambda_function" "handler" {
  function_name = "my-handler"
  role          = aws_iam_role.lambda_exec.arn

  depends_on = [time_sleep.wait_for_iam]
  # ...
}
```

## Strategy 2: depends_on for Correct Ordering

Sometimes adding a `depends_on` to the policy attachment (not just the role) is enough to ensure the attachment completes before the Lambda is created:

```hcl
resource "aws_lambda_function" "handler" {
  function_name = "my-handler"
  role          = aws_iam_role.lambda_exec.arn

  # Explicitly wait for the policy to be attached, not just the role created
  depends_on = [
    aws_iam_role_policy_attachment.lambda_exec,
    aws_iam_role_policy.inline_policy
  ]
}
```

## Strategy 3: Polling with null_resource

For cases where the propagation time is highly variable, poll until the operation succeeds:

```hcl
resource "null_resource" "wait_for_certificate" {
  triggers = {
    cert_arn = aws_acm_certificate.main.arn
  }

  provisioner "local-exec" {
    command = <<-EOT
      aws acm wait certificate-validated \
        --certificate-arn ${self.triggers.cert_arn} \
        --region us-east-1
    EOT
  }

  depends_on = [aws_acm_certificate_validation.main]
}
```

## Strategy 4: Provider-Specific Waiter Configuration

Some providers support custom waiters. For AWS, many resources implicitly wait for the resource to become available. Configure maximum wait times:

```hcl
resource "aws_elasticsearch_domain" "search" {
  domain_name = "my-search"

  timeouts {
    create = "60m"  # Elasticsearch creation can take up to an hour
    update = "60m"
    delete = "40m"
  }
}
```

## Strategy 5: Idempotent Data Sources with Polling

Use a data source with explicit `depends_on` to read a resource only after its dependencies are fully ready:

```hcl
# Create the bucket policy
resource "aws_s3_bucket_policy" "main" {
  bucket = aws_s3_bucket.main.id
  policy = data.aws_iam_policy_document.bucket.json
}

# Read back the bucket after policy is applied
data "aws_s3_bucket" "main" {
  bucket     = aws_s3_bucket.main.id
  depends_on = [aws_s3_bucket_policy.main]  # Wait for policy to propagate
}
```

## Diagnosing Eventual Consistency Failures

Look for these error patterns in your apply output:

```
Error: error creating Lambda function: InvalidParameterValueException:
The role defined for the function cannot be assumed by Lambda.
```

This is a classic IAM propagation error. The fix is always a `time_sleep` or `depends_on` between the IAM attachment and the Lambda creation.

## Conclusion

Eventual consistency is unavoidable in cloud infrastructure. Handle it in OpenTofu with `time_sleep` for fixed-duration waits, `depends_on` for ordering guarantees, `null_resource` polling for variable-duration waits, and provider timeout configuration for long-running operations. Match the strategy to the specific propagation behavior of the service causing the issue.
