# How to Configure IPv6 for AWS Lambda

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, IPv6, Lambda, Serverless, VPC, Function URLs

Description: Enable IPv6 for AWS Lambda functions, configure VPC-attached functions to use dual-stack subnets, and expose functions via IPv6-capable Function URLs or API Gateway.

## Introduction

AWS Lambda supports IPv6 in two contexts: Lambda Function URLs with dualstack mode (enabling direct IPv6 invocation), and VPC-connected Lambda functions that can use IPv6-enabled subnets to make outbound IPv6 connections. IPv6 in Lambda is particularly useful for functions that call IPv6-only endpoints or need to operate in IPv6-only VPC environments.

## Lambda Function URLs with IPv6

```bash
# Create a Lambda function URL with dualstack mode

FUNCTION_ARN="arn:aws:lambda:us-east-1:123456789:function:my-function"

aws lambda create-function-url-config \
    --function-name my-function \
    --auth-type NONE \
    --invoke-mode BUFFERED

# Enable dualstack on function URL (HTTP/2 with IPv6)
aws lambda update-function-url-config \
    --function-name my-function \
    --invoke-mode BUFFERED

# Get the function URL
aws lambda get-function-url-config \
    --function-name my-function \
    --query "FunctionUrl"

# Test IPv6 access to function URL
FUNCTION_URL="https://abc123.lambda-url.us-east-1.on.aws/"
curl -6 "$FUNCTION_URL"
```

## Terraform Lambda with IPv6

```hcl
# lambda_ipv6.tf

resource "aws_lambda_function" "api" {
  filename         = "function.zip"
  function_name    = "ipv6-api"
  role             = aws_iam_role.lambda_role.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = filebase64sha256("function.zip")

  # VPC configuration for IPv6 outbound
  vpc_config {
    subnet_ids = [
      aws_subnet.private_a.id,  # IPv6-enabled subnets
      aws_subnet.private_b.id,
    ]
    security_group_ids = [aws_security_group.lambda.id]
  }

  environment {
    variables = {
      STAGE = "production"
    }
  }

  tags = { Name = "ipv6-api-lambda" }
}

# Function URL for direct IPv6 invocation
resource "aws_lambda_function_url" "api" {
  function_name      = aws_lambda_function.api.function_name
  authorization_type = "AWS_IAM"

  # CORS configuration
  cors {
    allow_credentials = true
    allow_origins     = ["https://example.com"]
    allow_methods     = ["GET", "POST"]
    allow_headers     = ["Content-Type"]
    max_age           = 300
  }
}

# Security group for Lambda
resource "aws_security_group" "lambda" {
  vpc_id = aws_vpc.main.id
  name   = "lambda-sg"

  # Allow all outbound (IPv4 + IPv6)
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = { Name = "lambda-sg" }
}

output "function_url" {
  value = aws_lambda_function_url.api.function_url
}
```

## Lambda Function Making IPv6 Outbound Calls

```javascript
// Lambda function that makes IPv6 outbound requests
const https = require('https');

exports.handler = async (event) => {
    // This Lambda is in a VPC with IPv6-enabled subnets
    // and Egress-Only IGW for IPv6 outbound

    const options = {
        hostname: 'ipv6.icanhazip.com',
        port: 443,
        path: '/',
        method: 'GET',
        family: 6  // Force IPv6 (Node.js net.connect option)
    };

    return new Promise((resolve, reject) => {
        const req = https.request(options, (res) => {
            let data = '';
            res.on('data', (chunk) => data += chunk);
            res.on('end', () => {
                resolve({
                    statusCode: 200,
                    body: JSON.stringify({
                        ipv6_address: data.trim(),
                        message: 'Lambda made IPv6 outbound connection'
                    })
                });
            });
        });

        req.on('error', reject);
        req.end();
    });
};
```

## API Gateway with IPv6 (via CloudFront)

```bash
# API Gateway itself doesn't natively support IPv6 directly
# Use CloudFront in front of API Gateway for IPv6

# 1. Create API Gateway HTTP API
API_ID=$(aws apigatewayv2 create-api \
    --name ipv6-api \
    --protocol-type HTTP \
    --query "ApiId" \
    --output text)

# 2. Create CloudFront distribution pointing to API Gateway
# Set is_ipv6_enabled = true in CloudFront
# Use API Gateway invoke URL as origin
```

## Lambda VPC IPv6 Outbound

```bash
# For Lambda in VPC to make IPv6 outbound calls:
# 1. Lambda must be in IPv6-enabled subnet
# 2. Subnet route table must have ::/0 → EIGW
# 3. Lambda security group must allow IPv6 egress

# Verify Lambda can reach IPv6 endpoints
# Add test code to the function:
# const { exec } = require('child_process');
# exec('curl -6 https://ipv6.icanhazip.com', callback)
```

## Conclusion

Lambda IPv6 support comes through two paths: Function URLs that can be accessed over IPv6 directly (when in dualstack mode), and VPC-connected functions that can make outbound IPv6 connections through IPv6-enabled subnets with Egress-Only Internet Gateways. For public IPv6 access to API-style Lambda functions, put CloudFront (with `is_ipv6_enabled = true`) in front of Lambda Function URLs or API Gateway. Lambda functions themselves have no IPv4/IPv6 settings - IPv6 behavior depends entirely on the VPC and subnet configuration.
