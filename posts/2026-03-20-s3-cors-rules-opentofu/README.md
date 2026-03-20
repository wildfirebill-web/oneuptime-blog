# How to Configure S3 CORS Rules with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, S3, CORS, Web Applications, Infrastructure as Code, Frontend

Description: Learn how to configure S3 CORS (Cross-Origin Resource Sharing) rules using OpenTofu to allow web browsers to make cross-origin requests to S3 for static assets and direct uploads.

## Introduction

CORS rules allow web browsers to make requests to S3 buckets from different origins. This is required for browser-based applications that read from or write directly to S3, such as single-page applications using pre-signed URLs for direct uploads or fetching public assets.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with S3 permissions

## Step 1: Create an S3 Bucket for Web Assets

```hcl
resource "aws_s3_bucket" "web_assets" {
  bucket = "${var.project_name}-web-assets"
  tags   = { Name = "web-assets" }
}

resource "aws_s3_bucket_public_access_block" "web_assets" {
  bucket                  = aws_s3_bucket.web_assets.id
  block_public_acls       = true
  block_public_policy     = false  # Allow bucket policy for CloudFront
  ignore_public_acls      = true
  restrict_public_buckets = false
}
```

## Step 2: Configure CORS Rules

```hcl
resource "aws_s3_bucket_cors_configuration" "web_assets" {
  bucket = aws_s3_bucket.web_assets.id

  # Rule for reading static assets from the web app
  cors_rule {
    id              = "web-app-read"
    allowed_origins = ["https://app.example.com", "https://www.example.com"]
    allowed_methods = ["GET", "HEAD"]
    allowed_headers = ["*"]
    expose_headers  = ["ETag", "Content-Type", "Content-Length"]
    max_age_seconds = 86400  # Cache preflight response for 24 hours
  }

  # Rule for direct browser uploads using pre-signed URLs
  cors_rule {
    id              = "direct-upload"
    allowed_origins = ["https://app.example.com"]
    allowed_methods = ["PUT", "POST"]
    allowed_headers = [
      "Content-Type",
      "Content-MD5",
      "x-amz-server-side-encryption",
      "x-amz-meta-*"
    ]
    expose_headers  = ["ETag"]
    max_age_seconds = 3600
  }

  # Development rule (only enable in non-production!)
  dynamic "cors_rule" {
    for_each = var.environment == "dev" ? [1] : []
    content {
      id              = "dev-access"
      allowed_origins = ["http://localhost:3000", "http://localhost:8080"]
      allowed_methods = ["GET", "PUT", "POST", "DELETE", "HEAD"]
      allowed_headers = ["*"]
      expose_headers  = ["ETag"]
      max_age_seconds = 300
    }
  }
}
```

## Step 3: Configure a Pre-Signed URL Lambda for Uploads

```python
# Lambda function to generate pre-signed upload URLs

import boto3
import json
import os

s3 = boto3.client('s3')

def handler(event, context):
    """Generate a pre-signed URL for direct browser upload."""
    body = json.loads(event['body'])
    file_name = body['fileName']
    content_type = body['contentType']
    bucket = os.environ['BUCKET_NAME']

    # Generate pre-signed URL valid for 15 minutes
    url = s3.generate_presigned_url(
        'put_object',
        Params={
            'Bucket': bucket,
            'Key': f"uploads/{file_name}",
            'ContentType': content_type,
            'ServerSideEncryption': 'aws:kms'
        },
        ExpiresIn=900,
        HttpMethod='PUT'
    )

    return {
        'statusCode': 200,
        'headers': {
            'Access-Control-Allow-Origin': 'https://app.example.com',
            'Access-Control-Allow-Methods': 'POST',
        },
        'body': json.dumps({'uploadUrl': url, 'key': f"uploads/{file_name}"})
    }
```

## Step 4: Test CORS Configuration

```bash
# Test CORS preflight request
curl -X OPTIONS \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: PUT" \
  -H "Access-Control-Request-Headers: Content-Type" \
  -v https://my-bucket.s3.amazonaws.com/test.jpg 2>&1 | grep -i "access-control"
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

S3 CORS rules enable browser-based interactions with S3 for reading assets and direct uploads. Use specific `allowed_origins` rather than wildcards in production to prevent unauthorized cross-origin access. For direct browser uploads, generate pre-signed URLs server-side with short expiration times and include content-type restrictions to prevent clients from uploading unexpected file types.
