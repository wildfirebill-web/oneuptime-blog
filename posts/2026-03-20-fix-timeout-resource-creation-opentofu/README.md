# How to Fix Timeout Errors During Resource Creation in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Troubleshooting, Timeout, Resource Creation, Error, Infrastructure as Code

Description: Learn how to resolve timeout errors during resource creation in OpenTofu by increasing timeout values, investigating slow resource provisioning, and handling idempotent retries.

## Introduction

Timeout errors occur when a cloud resource takes longer to provision than the default timeout configured in the provider. Long-running resources like RDS instances, EKS clusters, and network appliances are common culprits.

## Common Timeout Error Messages

```text
Error: waiting for EC2 Transit Gateway (tgw-0abc123) to become available: timeout while waiting for state to become 'available' (last state: 'pending', timeout: 10m0s)

Error: waiting for RDS DB Instance (prod-postgres) to be created: timeout after 40m0s

Error: creating EKS Cluster (prod-eks): unexpected state 'CREATING', wanted target 'ACTIVE'. last error: %!s(<nil>)
```

## Fix 1: Override the Timeout in the Resource Block

Most providers support a `timeouts` block per resource:

```hcl
resource "aws_db_instance" "main" {
  identifier        = "prod-postgres"
  engine            = "postgres"
  instance_class    = "db.r5.large"
  allocated_storage = 100

  # Increase the create timeout for large instances
  timeouts {
    create = "60m"   # Default is often 40m - increase to 60m
    update = "80m"
    delete = "60m"
  }
}

resource "aws_eks_cluster" "main" {
  name     = "prod-eks"
  role_arn = aws_iam_role.eks.arn

  timeouts {
    create = "30m"
    update = "60m"
    delete = "15m"
  }

  vpc_config {
    subnet_ids = aws_subnet.private[*].id
  }
}
```

## Fix 2: Investigate Slow Provisioning

Before increasing timeouts, investigate why the resource is slow:

```bash
# Enable debug logging to see what the provider is waiting on

TF_LOG=DEBUG tofu apply 2>&1 | grep -iE "waiting|polling|status|state"

# For AWS: check resource status directly
aws rds describe-db-instances --db-instance-identifier prod-postgres \
  --query "DBInstances[0].DBInstanceStatus"

aws eks describe-cluster --name prod-eks \
  --query "cluster.status"
```

## Fix 3: Check Cloud Service Limits

Slow creation can be caused by account limits being reached:

```bash
# AWS: Check service quotas
aws service-quotas list-service-quotas --service-code rds \
  --query "Quotas[?QuotaName=='DB instances']"

# Request a quota increase if needed
aws service-quotas request-service-quota-increase \
  --service-code rds \
  --quota-code L-7B9F4748 \
  --desired-value 40
```

## Fix 4: Idempotent Retry After Timeout

If apply timed out but the resource was actually created (OpenTofu just didn't see it in time), re-running apply will import the resource automatically on the next plan:

```bash
# Run plan to see the current state
tofu plan

# If the resource exists and plan shows no changes, you're done
# If plan wants to recreate it, import it first
tofu import aws_db_instance.main prod-postgres
```

## Fix 5: Pre-Warm Resources That Take Long to Create

For clusters and databases with known long creation times, create them in a separate configuration that runs before the main apply:

```bash
# Stage 1: Create long-running resources
tofu apply -target=aws_eks_cluster.main
tofu apply -target=aws_db_instance.main

# Stage 2: Apply everything else that depends on them
tofu apply
```

## Fix 6: Split Large Applies

Very large configurations applying hundreds of resources simultaneously can cause timeouts due to API rate limiting:

```bash
# Use parallelism to limit concurrent resource creation
tofu apply -parallelism=5
```

## Conclusion

Timeout errors during resource creation are fixed by increasing the `timeouts` block values for slow-provisioning resources, investigating whether the underlying cloud service is genuinely slow (quotas, service degradation), and checking whether the resource was actually created (idempotent re-apply). For consistently slow resources, pre-warm them in a separate targeted apply before the main configuration.
