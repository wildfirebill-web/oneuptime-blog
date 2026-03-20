# How to Configure AWS API Gateway with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, API Gateway, IPv6, Cloud, Dualstack, Terraform

Description: Enable IPv6 access for AWS API Gateway HTTP and REST APIs using dualstack endpoints, custom domains, and CloudFront integration.

## Introduction

AWS API Gateway supports IPv6 through its dualstack endpoint mode introduced for HTTP APIs. REST APIs gain IPv6 access via custom domain names backed by CloudFront, which has been dualstack-enabled since 2021.

## HTTP API - Enable Dualstack Endpoint

AWS HTTP APIs (v2) support a built-in dualstack endpoint. Enable it at creation time or update an existing API.

```bash
# Create a new HTTP API with dualstack enabled

aws apigatewayv2 create-api \
  --name "MyIPv6API" \
  --protocol-type HTTP \
  --ip-address-type dualstack \
  --region us-east-1

# Update an existing HTTP API to dualstack
aws apigatewayv2 update-api \
  --api-id "abc123def" \
  --ip-address-type dualstack \
  --region us-east-1
```

After enabling dualstack, the default execute-api endpoint resolves both A and AAAA records.

## REST API - IPv6 via CloudFront Custom Domain

REST APIs do not have a native dualstack toggle, but a custom domain backed by CloudFront provides IPv6 access.

### Step 1: Create a Custom Domain in API Gateway

```bash
# Create a custom domain name with ACM certificate
aws apigateway create-domain-name \
  --domain-name api.example.com \
  --endpoint-configuration types=EDGE \
  --certificate-arn arn:aws:acm:us-east-1:123456789:certificate/abc-123
```

### Step 2: Enable Dualstack on the CloudFront Distribution

When API Gateway creates a CloudFront distribution for an EDGE endpoint, enable dualstack (IPv6) on it.

```bash
# Get the CloudFront distribution ID from the domain name
DIST_ID=$(aws apigateway get-domain-name \
  --domain-name api.example.com \
  --query 'distributionDomainName' \
  --output text | \
  xargs -I{} aws cloudfront list-distributions \
  --query "DistributionList.Items[?DomainName=='{}'].Id" \
  --output text)

# Enable IPv6 on the CloudFront distribution
aws cloudfront get-distribution-config \
  --id "$DIST_ID" > dist-config.json

# Edit dist-config.json to set "IsIPV6Enabled": true under DistributionConfig
# Then update:
aws cloudfront update-distribution \
  --id "$DIST_ID" \
  --distribution-config file://dist-config.json \
  --if-match $(jq -r '.ETag' dist-config.json)
```

## Terraform Example - HTTP API with Dualstack

```hcl
# main.tf - AWS HTTP API Gateway with IPv6 dualstack

resource "aws_apigatewayv2_api" "ipv6_api" {
  name          = "ipv6-http-api"
  protocol_type = "HTTP"

  # Enable both IPv4 and IPv6 endpoint types
  ip_address_type = "dualstack"
}

resource "aws_apigatewayv2_stage" "default" {
  api_id      = aws_apigatewayv2_api.ipv6_api.id
  name        = "$default"
  auto_deploy = true
}

output "api_endpoint" {
  value = aws_apigatewayv2_api.ipv6_api.api_endpoint
}
```

## Verify IPv6 Resolution

```bash
# Resolve the API endpoint for AAAA records
dig AAAA abc123def.execute-api.us-east-1.amazonaws.com

# Test the API over IPv6
curl -6 https://abc123def.execute-api.us-east-1.amazonaws.com/health

# Test custom domain over IPv6
curl -6 https://api.example.com/health
```

## Lambda Integration - Handle IPv6 Client IPs

When clients connect over IPv6, API Gateway passes the client address in `requestContext.identity.sourceIp`. Your Lambda function receives it correctly without changes.

```python
# Lambda handler - IPv6 source IP is in the event automatically
def handler(event, context):
    source_ip = event['requestContext']['identity']['sourceIp']
    # source_ip may be e.g. "2001:db8::1" for IPv6 clients
    print(f"Request from: {source_ip}")
    return {"statusCode": 200, "body": "OK"}
```

## Conclusion

AWS HTTP APIs have first-class dualstack support via `ip_address_type: dualstack`. REST APIs require a CloudFront-backed custom domain with IPv6 enabled on the distribution. Monitor your API Gateway endpoints from both IPv4 and IPv6 perspectives using OneUptime to ensure parity.
