# How to Use ElastiCache Redis with Lambda Functions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ElastiCache, AWS Lambda, Serverless, Caching

Description: Learn how to connect AWS Lambda functions to ElastiCache Redis using VPC configuration and connection reuse to minimize latency and cold-start overhead.

---

Lambda functions can connect to ElastiCache Redis, but because Lambda is stateless, you must handle connection management carefully. The key is reusing connections across invocations by initializing the client outside the handler.

## VPC Configuration

ElastiCache Redis is only accessible within a VPC. Your Lambda function must be in the same VPC (or a peered VPC) and have a security group that allows outbound access to the Redis security group.

```bash
# Allow Lambda SG to reach Redis SG on port 6379
aws ec2 authorize-security-group-ingress \
  --group-id sg-redis \
  --protocol tcp \
  --port 6379 \
  --source-group sg-lambda
```

## Lambda IAM Role

No IAM permissions are needed to call Redis directly - authentication uses the auth token. But you do need VPC permissions:

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:CreateNetworkInterface",
    "ec2:DescribeNetworkInterfaces",
    "ec2:DeleteNetworkInterface"
  ],
  "Resource": "*"
}
```

## Connection Reuse Pattern

Initialize the Redis client outside the handler so it persists across warm invocations:

```python
import redis
import json
import os

# Module-level client - reused across invocations
_redis_client = None

def get_client():
    global _redis_client
    if _redis_client is None:
        _redis_client = redis.Redis(
            host=os.environ["REDIS_HOST"],
            port=6379,
            ssl=True,
            password=os.environ.get("REDIS_AUTH_TOKEN"),
            socket_connect_timeout=2,
            socket_timeout=2,
            decode_responses=True,
        )
    return _redis_client

def lambda_handler(event, context):
    r = get_client()
    user_id = event.get("user_id")
    cache_key = f"user:{user_id}:profile"

    cached = r.get(cache_key)
    if cached:
        return {"source": "cache", "data": json.loads(cached)}

    # Simulate DB call
    profile = {"id": user_id, "name": "Alice"}
    r.setex(cache_key, 300, json.dumps(profile))
    return {"source": "database", "data": profile}
```

## Environment Variables

```bash
aws lambda update-function-configuration \
  --function-name my-function \
  --environment Variables='{
    "REDIS_HOST":"my-cluster.abc.cache.amazonaws.com",
    "REDIS_AUTH_TOKEN":"MyStr0ngP@ssword"
  }'
```

## Terraform Lambda with VPC

```hcl
resource "aws_lambda_function" "api" {
  function_name = "my-function"
  runtime       = "python3.12"
  handler       = "handler.lambda_handler"
  role          = aws_iam_role.lambda.arn
  filename      = "function.zip"

  vpc_config {
    subnet_ids         = var.private_subnet_ids
    security_group_ids = [aws_security_group.lambda.id]
  }

  environment {
    variables = {
      REDIS_HOST       = aws_elasticache_replication_group.main.primary_endpoint_address
      REDIS_AUTH_TOKEN = var.redis_auth_token
    }
  }
}
```

## Handling Cold Starts

Cold starts add 200-500ms for VPC-attached Lambda. Use Provisioned Concurrency for latency-sensitive functions:

```bash
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier LIVE \
  --provisioned-concurrent-executions 5
```

## Summary

Connect Lambda to ElastiCache Redis by placing both in the same VPC with matching security group rules. Initialize the Redis client at module level (outside the handler) to reuse connections across warm invocations. Use Provisioned Concurrency for latency-sensitive functions to avoid VPC cold-start delays, and always use TLS with an auth token.
