# How to Implement Retry Logic with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Retry Logic, Resilience, Provisioner, Infrastructure as Code

Description: Learn how to implement retry logic in OpenTofu for transient failures using provider settings, null_resource retries, and external_provider patterns.

Cloud APIs fail transiently - rate limits, eventually-consistent reads, and service-side timeouts are routine. OpenTofu itself does not have a built-in retry keyword for resource operations, but several patterns let you build retry behavior into your configurations.

## Provider-Level Retry Settings

The first line of defense is configuring the provider's built-in retry mechanism:

```hcl
# AWS: increase API retry count and use adaptive retry mode

provider "aws" {
  region      = "us-east-1"
  retry_mode  = "adaptive"   # Automatically backs off on throttling
  max_retries = 10           # Default is 3
}
```

```hcl
# Google Cloud: configure custom timeout per operation
provider "google" {
  project = var.project_id
  region  = "us-central1"
}

# Each resource supports timeout blocks
resource "google_container_cluster" "primary" {
  name = "my-cluster"

  timeouts {
    create = "30m"  # Wait up to 30 minutes before failing
    update = "30m"
    delete = "30m"
  }
}
```

Resource-Level Timeouts

Most resources support a `timeouts` block for controlling how long OpenTofu waits for operations:

```hcl
resource "aws_db_instance" "main" {
  identifier     = "myapp-db"
  engine         = "postgres"
  instance_class = "db.t3.medium"
  # ...

  timeouts {
    create = "40m"  # RDS creation can take a while
    update = "40m"
    delete = "40m"
  }
}
```

## Retry with null_resource and local-exec

For operations that require retry logic not supported natively by providers, use `null_resource` with a retry loop in a provisioner:

```hcl
resource "null_resource" "wait_for_service" {
  # Re-run when the service endpoint changes
  triggers = {
    endpoint = aws_lb.app.dns_name
  }

  provisioner "local-exec" {
    # Retry health check up to 30 times with 10-second intervals
    command = <<-EOT
      for i in $(seq 1 30); do
        if curl -sf "http://${self.triggers.endpoint}/health"; then
          echo "Service is healthy"
          exit 0
        fi
        echo "Attempt $i/30 failed, retrying in 10s..."
        sleep 10
      done
      echo "Service health check timed out"
      exit 1
    EOT
  }

  depends_on = [aws_lb.app]
}
```

## Retry with time_sleep for Eventual Consistency

Use the `time_sleep` resource to add a wait between dependent operations:

```hcl
resource "time_sleep" "wait_for_iam_propagation" {
  depends_on = [aws_iam_role_policy_attachment.lambda]

  # IAM changes typically propagate within 10-15 seconds globally
  create_duration = "20s"
}

resource "aws_lambda_function" "processor" {
  depends_on = [time_sleep.wait_for_iam_propagation]
  # ...
}
```

## Retry Wrapper Script for CI Pipelines

Wrap `tofu apply` in a retry loop in your CI pipeline for transient failures:

```bash
#!/usr/bin/env bash
# retry-apply.sh - retry tofu apply on transient failures

set -euo pipefail

MAX_ATTEMPTS=3
RETRY_DELAY=30

for attempt in $(seq 1 $MAX_ATTEMPTS); do
  echo "Apply attempt $attempt/$MAX_ATTEMPTS..."

  if tofu apply -auto-approve -input=false; then
    echo "Apply succeeded on attempt $attempt."
    exit 0
  fi

  if [[ $attempt -lt $MAX_ATTEMPTS ]]; then
    echo "Apply failed. Retrying in ${RETRY_DELAY}s..."
    sleep $RETRY_DELAY
  fi
done

echo "Apply failed after $MAX_ATTEMPTS attempts."
exit 1
```

## Using create_before_destroy for Zero-Downtime Replacements

When a resource must be replaced, `create_before_destroy` ensures the new instance is ready before the old one is deleted:

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true
  }
}
```

## Conclusion

Retry logic in OpenTofu is implemented at multiple layers: provider retry settings for API-level throttling, `timeouts` blocks for long-running operations, `null_resource` with shell loops for application-level health checks, `time_sleep` for eventual consistency waits, and CI-level retry wrappers for transient apply failures. Use the appropriate layer for each type of failure.
