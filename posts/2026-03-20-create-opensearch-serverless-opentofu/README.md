# How to Create AWS OpenSearch Serverless with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, OpenSearch Serverless, Search, Infrastructure as Code

Description: Learn how to create AWS OpenSearch Serverless collections with OpenTofu for on-demand search and analytics without cluster management.

OpenSearch Serverless removes cluster capacity planning by automatically scaling compute and storage. Managing collections, security policies, and data policies in OpenTofu ensures consistent configuration and access control.

## Creating a Collection

```hcl
resource "aws_opensearchserverless_collection" "logs" {
  name        = "application-logs"
  description = "Application log storage and search"
  type        = "TIMESERIES"  # SEARCH, TIMESERIES, or VECTORSEARCH

  tags = {
    Environment = "production"
    Team        = "data-platform"
  }

  depends_on = [
    aws_opensearchserverless_security_policy.encryption,
    aws_opensearchserverless_security_policy.network,
  ]
}
```

## Encryption Policy

```hcl
resource "aws_opensearchserverless_security_policy" "encryption" {
  name        = "logs-encryption-policy"
  type        = "encryption"
  description = "Encryption policy for logs collection"

  policy = jsonencode({
    Rules = [
      {
        Resource = ["collection/application-logs"]
        ResourceType = "collection"
      }
    ]
    AWSOwnedKey = true  # Use AWS-managed key; replace with KMS key ARN for CMK
  })
}
```

## Network Policy

```hcl
resource "aws_opensearchserverless_security_policy" "network" {
  name        = "logs-network-policy"
  type        = "network"
  description = "Network policy for logs collection"

  policy = jsonencode([
    {
      Rules = [
        {
          Resource     = ["collection/application-logs"]
          ResourceType = "collection"
        },
        {
          Resource     = ["collection/application-logs"]
          ResourceType = "dashboard"
        }
      ]
      AllowFromPublic = false
      SourceVPCEs     = [aws_opensearchserverless_vpc_endpoint.main.id]
    }
  ])
}
```

## Data Access Policy

```hcl
resource "aws_opensearchserverless_access_policy" "logs" {
  name        = "logs-access-policy"
  type        = "data"
  description = "Data access policy for logs collection"

  policy = jsonencode([
    {
      Rules = [
        {
          Resource     = ["index/application-logs/*"]
          ResourceType = "index"
          Permission   = [
            "aoss:CreateIndex",
            "aoss:DeleteIndex",
            "aoss:UpdateIndex",
            "aoss:DescribeIndex",
            "aoss:ReadDocument",
            "aoss:WriteDocument",
          ]
        },
        {
          Resource     = ["collection/application-logs"]
          ResourceType = "collection"
          Permission   = ["aoss:DescribeCollectionItems"]
        }
      ]
      Principal = [
        aws_iam_role.app.arn,
        aws_iam_role.log_shipper.arn,
      ]
    }
  ])
}
```

## VPC Endpoint

```hcl
resource "aws_opensearchserverless_vpc_endpoint" "main" {
  name               = "logs-vpc-endpoint"
  vpc_id             = var.vpc_id
  subnet_ids         = var.private_subnet_ids
  security_group_ids = [aws_security_group.opensearch.id]
}
```

## Multiple Collections

```hcl
locals {
  collections = {
    logs    = { type = "TIMESERIES", description = "Application and system logs" }
    search  = { type = "SEARCH",    description = "Product catalog search" }
    vectors = { type = "VECTORSEARCH", description = "ML embedding search" }
  }
}

resource "aws_opensearchserverless_security_policy" "encryption_policies" {
  for_each = local.collections

  name = "${each.key}-encryption"
  type = "encryption"

  policy = jsonencode({
    Rules = [
      {
        Resource     = ["collection/${each.key}"]
        ResourceType = "collection"
      }
    ]
    AWSOwnedKey = true
  })
}

resource "aws_opensearchserverless_collection" "collections" {
  for_each = local.collections

  name        = each.key
  description = each.value.description
  type        = each.value.type

  depends_on = [aws_opensearchserverless_security_policy.encryption_policies]
}
```

## Outputs

```hcl
output "collection_endpoint" {
  value = aws_opensearchserverless_collection.logs.collection_endpoint
}

output "dashboard_endpoint" {
  value = aws_opensearchserverless_collection.logs.dashboard_endpoint
}
```

## Conclusion

OpenSearch Serverless in OpenTofu eliminates cluster management while providing full-text search capabilities. Always create encryption and network policies before the collection (use depends_on), define data access policies to grant granular index-level permissions, and use VPC endpoints to keep traffic private. Use TIMESERIES for log analytics, SEARCH for document search, and VECTORSEARCH for embedding-based similarity search.
