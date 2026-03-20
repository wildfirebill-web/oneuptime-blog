# How to Configure DynamoDB Global Secondary Indexes with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, DynamoDB, Global Secondary Indexes, GSI, Query Performance, Infrastructure as Code

Description: Learn how to configure DynamoDB Global Secondary Indexes (GSIs) with OpenTofu to enable efficient queries on non-primary-key attributes without table scans.

## Introduction

DynamoDB Global Secondary Indexes (GSIs) allow you to query data using attributes other than the table's primary key. A GSI has its own partition key and optional sort key, and can include any subset of table attributes as projected attributes. You can add up to 20 GSIs per table, and each GSI can have its own read/write capacity. GSIs are eventually consistent by default and replicate data asynchronously from the base table.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with DynamoDB permissions

## Step 1: Create Table with GSI

```hcl
resource "aws_dynamodb_table" "orders" {
  name         = "${var.project_name}-orders"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "orderId"
  range_key    = "createdAt"

  attribute {
    name = "orderId"
    type = "S"
  }

  attribute {
    name = "createdAt"
    type = "S"
  }

  attribute {
    name = "customerId"
    type = "S"
  }

  attribute {
    name = "status"
    type = "S"
  }

  # GSI: Query orders by customer
  global_secondary_index {
    name            = "CustomerOrders"
    hash_key        = "customerId"
    range_key       = "createdAt"
    projection_type = "ALL"  # ALL, KEYS_ONLY, or INCLUDE
  }

  # GSI: Query orders by status
  global_secondary_index {
    name            = "OrdersByStatus"
    hash_key        = "status"
    range_key       = "createdAt"
    projection_type = "INCLUDE"
    non_key_attributes = ["orderId", "customerId", "totalAmount"]
  }

  tags = {
    Name = "${var.project_name}-orders"
  }
}
```

## Step 2: GSI with Provisioned Capacity

```hcl
resource "aws_dynamodb_table" "products" {
  name           = "${var.project_name}-products"
  billing_mode   = "PROVISIONED"
  read_capacity  = 10
  write_capacity = 5
  hash_key       = "productId"

  attribute {
    name = "productId"
    type = "S"
  }

  attribute {
    name = "category"
    type = "S"
  }

  attribute {
    name = "price"
    type = "N"
  }

  # GSI with separate read/write capacity
  global_secondary_index {
    name            = "CategoryPrice"
    hash_key        = "category"
    range_key       = "price"
    projection_type = "ALL"
    read_capacity   = 5  # Independent capacity from base table
    write_capacity  = 2
  }
}
```

## Step 3: Deploy

```bash
tofu init
tofu plan
tofu apply

# Query using GSI

aws dynamodb query \
  --table-name <table-name> \
  --index-name CustomerOrders \
  --key-condition-expression "customerId = :cid" \
  --expression-attribute-values '{":cid": {"S": "CUST-001"}}'
```

## Conclusion

Choose `projection_type = "ALL"` when you frequently access many attributes from GSI query results-it avoids costly table fetches but doubles storage. Use `INCLUDE` to project only the attributes your queries need, and `KEYS_ONLY` for aggregate queries that only need IDs. GSI writes consume additional write capacity since every table write that touches indexed attributes also writes to the GSI. Design your GSI partition keys to distribute load evenly-hot partition keys on GSIs cause the same throttling issues as on base tables.
