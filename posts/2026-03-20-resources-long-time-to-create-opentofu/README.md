# How to Handle Resources That Take a Long Time to Create in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Timeout, AWS, RDS, Infrastructure as Code, Best Practices

Description: Learn how to configure timeouts, use depends_on, and implement retry logic for OpenTofu resources that take a long time to provision.

## Introduction

Some AWS resources like RDS instances, EKS clusters, and Direct Connect attachments can take 15-60 minutes to provision. Without proper timeout and dependency configuration, OpenTofu may give up or apply resources in the wrong order. This guide covers patterns for handling slow resources.

## Setting Custom Timeouts

Most resources support `timeouts` blocks to override defaults.

```hcl
resource "aws_db_instance" "main" {
  identifier     = "${var.app_name}-db"
  engine         = "postgres"
  engine_version = "16.2"
  instance_class = "db.r6g.xlarge"
  allocated_storage = 100

  username = var.db_username
  password = var.db_password

  multi_az              = true
  db_subnet_group_name  = aws_db_subnet_group.main.name

  # Extend timeouts for large/multi-AZ instances
  timeouts {
    create = "60m"  # default is 40m
    update = "80m"  # major upgrades can take longer
    delete = "60m"
  }

  tags = {
    ManagedBy = "opentofu"
  }
}
```

## EKS Cluster Timeouts

```hcl
resource "aws_eks_cluster" "main" {
  name     = "${var.app_name}-cluster"
  role_arn = aws_iam_role.eks.arn
  version  = "1.30"

  vpc_config {
    subnet_ids = var.private_subnet_ids
  }

  timeouts {
    create = "30m"
    delete = "20m"
    update = "60m"  # upgrades take longer
  }
}
```

## Using depends_on to Sequence Slow Resources

Some resources appear created in state but aren't functionally ready yet. Use `depends_on` to enforce ordering.

```hcl
# EKS cluster must be fully ready before the node group

resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "main"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.private_subnet_ids

  scaling_config {
    desired_size = 2
    max_size     = 5
    min_size     = 1
  }

  # Wait for add-ons that depend on the cluster being fully operational
  depends_on = [
    aws_iam_role_policy_attachment.node_AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.node_AmazonEC2ContainerRegistryReadOnly,
  ]

  timeouts {
    create = "30m"
    update = "30m"
    delete = "30m"
  }
}
```

## Polling with null_resource

When a resource doesn't have built-in timeout support, use `null_resource` with a polling loop.

```hcl
resource "null_resource" "wait_for_endpoint" {
  depends_on = [aws_db_instance.main]

  provisioner "local-exec" {
    command = <<-SCRIPT
      # Poll until the endpoint is accepting connections
      MAX_ATTEMPTS=30
      ATTEMPT=0
      until pg_isready -h ${aws_db_instance.main.address} -p 5432 -U ${var.db_username}; do
        ATTEMPT=$((ATTEMPT + 1))
        if [ $ATTEMPT -ge $MAX_ATTEMPTS ]; then
          echo "Database did not become ready after $MAX_ATTEMPTS attempts"
          exit 1
        fi
        echo "Waiting for database... attempt $ATTEMPT/$MAX_ATTEMPTS"
        sleep 10
      done
      echo "Database is ready!"
    SCRIPT
  }
}

# Resources that need the DB to be fully ready wait on this null_resource
resource "null_resource" "run_migrations" {
  depends_on = [null_resource.wait_for_endpoint]

  provisioner "local-exec" {
    command = "flyway -url=jdbc:postgresql://${aws_db_instance.main.endpoint}/${var.db_name} migrate"
  }
}
```

## Parallel Creation for Independence

When resources don't depend on each other, OpenTofu creates them in parallel by default. Structure your configuration to maximize parallelism.

```hcl
# These will be created in parallel since they're independent
resource "aws_db_instance" "main" { ... }
resource "aws_elasticache_replication_group" "cache" { ... }
resource "aws_opensearch_domain" "search" { ... }

# This waits for all three
resource "null_resource" "all_datastores_ready" {
  depends_on = [
    aws_db_instance.main,
    aws_elasticache_replication_group.cache,
    aws_opensearch_domain.search,
  ]
}
```

## Summary

Handling slow resources in OpenTofu requires extending timeout blocks, using `depends_on` for functional ordering, and polling with `null_resource` for resources without built-in readiness checks. Parallel independent resource creation reduces total wait time for complex stacks.
