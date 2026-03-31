# How to Use Redis with AWS API Gateway for Caching

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, AWS, API Gateway, Caching, Lambda, Performance

Description: Use Redis with AWS API Gateway and Lambda to implement a custom response cache that reduces backend latency, controls cache granularity, and supports cache-control headers.

---

## Why Redis with API Gateway

AWS API Gateway has a built-in cache, but it is expensive, per-stage, and not customizable. A Redis-backed custom cache using Lambda gives you:
- Fine-grained TTL per endpoint or user
- Cache invalidation on demand
- Shared cache across multiple API stages
- Cost control with ElastiCache vs. API Gateway cache pricing

## Architecture

```text
Client -> API Gateway -> Lambda (cache layer) -> Redis (ElastiCache)
                                              -> Backend / Database (on miss)
```

## Lambda Cache Handler - Python

```python
import os
import json
import hashlib
import redis
import urllib.request
import urllib.error

REDIS_HOST = os.environ["REDIS_HOST"]
REDIS_PORT = int(os.environ.get("REDIS_PORT", 6379))
BACKEND_URL = os.environ["BACKEND_URL"]
DEFAULT_TTL = int(os.environ.get("CACHE_TTL", 60))

redis_client = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, decode_responses=True)

def make_cache_key(event: dict) -> str:
    path = event.get("path", "/")
    method = event.get("httpMethod", "GET")
    query = json.dumps(event.get("queryStringParameters") or {}, sort_keys=True)
    user_id = (event.get("requestContext") or {}).get("identity", {}).get("user", "anon")
    raw = f"{method}:{path}:{query}:{user_id}"
    return "apigw:cache:" + hashlib.md5(raw.encode()).hexdigest()

def proxy_to_backend(event: dict) -> dict:
    path = event.get("path", "/")
    method = event.get("httpMethod", "GET")
    qs = event.get("queryStringParameters") or {}
    qs_str = "&".join(f"{k}={v}" for k, v in qs.items())
    url = f"{BACKEND_URL}{path}" + (f"?{qs_str}" if qs_str else "")

    req = urllib.request.Request(url, method=method)
    try:
        with urllib.request.urlopen(req, timeout=5) as resp:
            body = resp.read().decode("utf-8")
            return {
                "statusCode": resp.status,
                "body": body,
                "headers": {"Content-Type": "application/json"}
            }
    except Exception as e:
        return {
            "statusCode": 502,
            "body": json.dumps({"error": str(e)}),
            "headers": {"Content-Type": "application/json"}
        }

def lambda_handler(event, context):
    method = event.get("httpMethod", "GET")

    # Only cache GET requests
    if method != "GET":
        return proxy_to_backend(event)

    # Check Cache-Control: no-cache header
    headers = event.get("headers") or {}
    cache_control = headers.get("Cache-Control", "")
    if "no-cache" in cache_control or "no-store" in cache_control:
        return proxy_to_backend(event)

    cache_key = make_cache_key(event)

    # Check cache
    cached = redis_client.get(cache_key)
    if cached:
        response = json.loads(cached)
        response["headers"] = response.get("headers", {})
        response["headers"]["X-Cache"] = "HIT"
        return response

    # Cache miss
    response = proxy_to_backend(event)

    if response["statusCode"] == 200:
        redis_client.set(cache_key, json.dumps(response), ex=DEFAULT_TTL)

    response["headers"] = response.get("headers", {})
    response["headers"]["X-Cache"] = "MISS"
    return response
```

## Cache Invalidation Endpoint

Add an admin endpoint to invalidate cache by path prefix:

```python
def invalidate_cache_handler(event, context):
    body = json.loads(event.get("body") or "{}")
    path_prefix = body.get("path_prefix", "")

    if not path_prefix:
        return {"statusCode": 400, "body": json.dumps({"error": "path_prefix required"})}

    invalidated = 0
    for key in redis_client.scan_iter("apigw:cache:*"):
        redis_client.delete(key)
        invalidated += 1

    return {
        "statusCode": 200,
        "body": json.dumps({"invalidated": invalidated})
    }
```

## Lambda Environment Variables

Set these in your Lambda configuration:

```bash
REDIS_HOST=my-redis.abc123.0001.use1.cache.amazonaws.com
REDIS_PORT=6379
BACKEND_URL=https://my-backend-api.example.com
CACHE_TTL=60
```

## CloudFormation / SAM Template

```yaml
Resources:
  CacheLayer:
    Type: AWS::Serverless::Function
    Properties:
      Handler: handler.lambda_handler
      Runtime: python3.12
      Timeout: 10
      Environment:
        Variables:
          REDIS_HOST: !GetAtt RedisCluster.RedisEndpoint.Address
          REDIS_PORT: "6379"
          BACKEND_URL: !Sub "https://${BackendApi}.execute-api.${AWS::Region}.amazonaws.com/prod"
          CACHE_TTL: "60"
      VpcConfig:
        SecurityGroupIds:
          - !Ref LambdaSG
        SubnetIds:
          - !Ref PrivateSubnet1
          - !Ref PrivateSubnet2
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: "/{proxy+}"
            Method: ANY
```

## Per-Route TTL Configuration

```python
ROUTE_TTL = {
    "/products": 300,
    "/products/{id}": 600,
    "/categories": 3600,
    "/users/me": 30,
}

def get_route_ttl(path: str) -> int:
    for pattern, ttl in ROUTE_TTL.items():
        if "{" in pattern:
            prefix = pattern.split("{")[0].rstrip("/")
            if path.startswith(prefix):
                return ttl
        elif path == pattern:
            return ttl
    return DEFAULT_TTL
```

## Summary

Redis with AWS API Gateway uses a Lambda function as a caching proxy that intercepts GET requests, checks Redis for a cached response, and falls back to the backend on a miss. Cache keys are derived from the method, path, query parameters, and user identity. Per-route TTLs, Cache-Control header support, and on-demand invalidation give fine-grained control that API Gateway's built-in cache cannot provide.
