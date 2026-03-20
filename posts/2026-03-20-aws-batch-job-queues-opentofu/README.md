# How to Create AWS Batch Job Queues with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Batch, Job Queues, HPC, Infrastructure as Code

Description: Learn how to create AWS Batch job queues, configure priority-based scheduling, and define job definitions using OpenTofu.

## Introduction

AWS Batch Job Queues hold submitted batch jobs until compute capacity is available in the associated compute environments. Multiple queues with different priorities allow you to control job scheduling order. OpenTofu manages queues, priorities, and job definitions as code.

## Creating a Job Queue

```hcl
resource "aws_batch_job_queue" "standard" {
  name     = "${var.app_name}-standard-${var.environment}"
  state    = "ENABLED"
  priority = 10  # lower number = lower priority

  # Associate with compute environments in order of priority
  compute_environment_order {
    order               = 1
    compute_environment = aws_batch_compute_environment.ec2.arn
  }

  tags = {
    Environment = var.environment
    ManagedBy   = "opentofu"
  }
}

# High-priority queue for urgent jobs
resource "aws_batch_job_queue" "priority" {
  name     = "${var.app_name}-priority-${var.environment}"
  state    = "ENABLED"
  priority = 100  # higher priority than standard queue

  compute_environment_order {
    order               = 1
    compute_environment = aws_batch_compute_environment.ec2.arn  # try on-demand first
  }

  compute_environment_order {
    order               = 2
    compute_environment = aws_batch_compute_environment.spot.arn  # fall back to spot
  }
}
```

## Job Definition

```hcl
resource "aws_batch_job_definition" "data_processor" {
  name = "${var.app_name}-data-processor"
  type = "container"

  container_properties = jsonencode({
    image      = "${var.ecr_repo}/data-processor:latest"
    vcpus      = 4
    memory     = 8192  # MB
    jobRoleArn = aws_iam_role.batch_job.arn

    environment = [
      {
        name  = "ENV"
        value = var.environment
      },
      {
        name  = "S3_BUCKET"
        value = var.data_bucket
      }
    ]

    mountPoints = [
      {
        containerPath = "/tmp"
        sourceVolume  = "tmp"
        readOnly      = false
      }
    ]

    volumes = [
      {
        name = "tmp"
        host = { sourcePath = "/tmp" }
      }
    ]

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.batch.name
        "awslogs-region"        = var.region
        "awslogs-stream-prefix" = "data-processor"
      }
    }
  })

  retry_strategy {
    attempts = 3

    evaluate_on_exit {
      on_status_reason = "Host EC2*"
      action           = "RETRY"  # retry if spot instance was terminated
    }

    evaluate_on_exit {
      on_reason    = "*"
      on_exit_code = "1"
      action       = "FAILED"
    }
  }

  timeout {
    attempt_duration_seconds = 3600  # 1 hour max
  }
}
```

## Job IAM Role

```hcl
resource "aws_iam_role" "batch_job" {
  name = "${var.app_name}-batch-job-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "batch_job_s3" {
  role = aws_iam_role.batch_job.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = "arn:aws:s3:::${var.data_bucket}/*"
    }]
  })
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

AWS Batch job queues with priority ordering and multiple compute environment fallbacks give you fine-grained control over batch job scheduling. OpenTofu manages queues, job definitions, retry strategies, and IAM roles together as a complete batch processing stack.
