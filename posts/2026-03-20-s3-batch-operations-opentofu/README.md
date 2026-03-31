# How to Set Up S3 Batch Operations with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, S3, Batch Operation, Bulk Processing, Lambda, Infrastructure as Code

Description: Learn how to set up S3 Batch Operations using OpenTofu to perform large-scale bulk actions on millions of S3 objects including copying, tagging, and running custom Lambda functions.

## Introduction

S3 Batch Operations executes operations on billions of objects in parallel-far faster than iterating through them individually. Common use cases include copying objects across regions, restoring archived objects, updating object tags, ACLs, and running custom Lambda functions against every object.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with S3 Batch Operations and IAM permissions
- An inventory CSV file or S3 Inventory report

## Step 1: Create IAM Role for Batch Operations

```hcl
# IAM role that S3 Batch Operations assumes to perform actions

resource "aws_iam_role" "batch_ops" {
  name = "s3-batch-operations-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "batchoperations.s3.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "batch_ops" {
  name = "batch-operations-policy"
  role = aws_iam_role.batch_ops.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:GetObjectVersion",
          "s3:PutObject",
          "s3:CopyObject",
          "s3:PutObjectTagging"
        ]
        Resource = [
          "${var.source_bucket_arn}/*",
          "${var.destination_bucket_arn}/*"
        ]
      },
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:GetObjectVersion"]
        Resource = "${var.manifest_bucket_arn}/*"
      },
      {
        Effect   = "Allow"
        Action   = "s3:PutObject"
        Resource = "${var.report_bucket_arn}/*"
      },
      {
        Effect   = "Allow"
        Action   = "lambda:InvokeFunction"
        Resource = var.lambda_function_arn
      }
    ]
  })
}
```

## Step 2: Enable S3 Inventory for the Source Bucket

```hcl
# S3 Inventory generates a manifest for Batch Operations jobs
resource "aws_s3_bucket_inventory" "source" {
  bucket = var.source_bucket_name
  name   = "full-inventory"

  included_object_versions = "Current"  # Or "All" for versioned buckets

  schedule {
    frequency = "Daily"
  }

  destination {
    bucket {
      format     = "CSV"
      bucket_arn = var.inventory_bucket_arn
      prefix     = "inventory/${var.source_bucket_name}"
    }
  }

  optional_fields = [
    "Size", "LastModifiedDate", "StorageClass",
    "ETag", "IsMultipartUploaded", "EncryptionStatus",
    "ObjectLockMode", "ObjectLockRetainUntilDate"
  ]
}
```

## Step 3: Create a Batch Operations Job

```hcl
# S3 Batch Operations job to copy objects with new storage class
resource "aws_s3control_object_lambda_access_point" "copy_job" {
  # Note: Batch Operations jobs are created via AWS CLI or SDK
  # as the OpenTofu resource for batch jobs is limited
  # Use null_resource with CLI for job creation
  count = 0  # Placeholder
}

# Create the batch job via AWS CLI
resource "null_resource" "batch_copy_job" {
  triggers = {
    manifest_key    = var.manifest_key
    source_bucket   = var.source_bucket_arn
    dest_bucket     = var.destination_bucket_arn
  }

  provisioner "local-exec" {
    command = <<-EOF
      aws s3control create-job \
        --account-id ${data.aws_caller_identity.current.account_id} \
        --manifest '{"Spec":{"Format":"S3BatchOperations_CSV_20180820","Fields":["Bucket","Key"]},"Location":{"ObjectArn":"${var.manifest_bucket_arn}/${var.manifest_key}","ETag":"${var.manifest_etag}"}}' \
        --operation '{"S3CopyObject":{"TargetResource":"${var.destination_bucket_arn}","NewObjectMetadata":{},"StorageClass":"STANDARD_IA"}}' \
        --report '{"Bucket":"${var.report_bucket_arn}","Format":"Report_CSV_20180820","Enabled":true,"Prefix":"batch-reports","ReportScope":"AllTasks"}' \
        --priority 10 \
        --role-arn ${aws_iam_role.batch_ops.arn} \
        --region ${var.region} \
        --no-confirmation-required
    EOF
  }
}
```

## Step 4: Lambda-Based Custom Batch Operation

```python
# Lambda function for custom S3 Batch Operations processing
import boto3
import json

s3 = boto3.client('s3')

def handler(event, context):
    """Process each object in the batch job."""
    invocation_id = event['invocationId']
    tasks = event['tasks']
    results = []

    for task in tasks:
        task_id = task['taskId']
        bucket = task['s3BucketArn'].split(':::')[1]
        key = task['s3Key']

        try:
            # Custom processing: tag objects for lifecycle transition
            s3.put_object_tagging(
                Bucket=bucket,
                Key=key,
                Tagging={
                    'TagSet': [
                        {'Key': 'processed', 'Value': 'true'},
                        {'Key': 'processedDate', 'Value': '2026-03-20'}
                    ]
                }
            )
            results.append({
                'taskId': task_id,
                'resultCode': 'Succeeded',
                'resultString': 'Tags applied'
            })
        except Exception as e:
            results.append({
                'taskId': task_id,
                'resultCode': 'PermanentFailure',
                'resultString': str(e)
            })

    return {
        'invocationSchemaVersion': '1.0',
        'treatMissingKeysAs': 'PermanentFailure',
        'invocationId': invocation_id,
        'results': results
    }
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

S3 Batch Operations enables operations at massive scale that would take weeks using custom scripts in minutes. Use S3 Inventory to generate job manifests automatically, and enable job completion reports to track success rates. For custom processing like encryption re-encryption or metadata updates, Lambda-based jobs provide full flexibility while leveraging S3's managed parallelism.
