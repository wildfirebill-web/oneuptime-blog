# How to Cache gRPC Responses with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, gRPC, Caching

Description: Learn how to cache gRPC unary and server-streaming responses in Redis to reduce latency and backend load using protobuf serialization for compact storage.

---

gRPC responses are serialized protobuf binary data. Caching them in Redis is more efficient than caching JSON - the binary format is smaller and requires no re-serialization on cache hit. This guide covers caching unary RPCs and handling cache invalidation.

## Why Cache gRPC Responses

- Protobuf messages are compact - cache entries use less memory than equivalent JSON
- Unary RPCs for read-heavy operations (product lookups, user profiles) benefit most
- Backend protection: a hot key in Redis prevents thundering herd on the database

## Caching Unary RPC Responses (Python)

```python
import grpc
import redis
import hashlib
import json
import os

from concurrent import futures
import product_pb2
import product_pb2_grpc

r = redis.Redis.from_url(os.environ['REDIS_URL'])

class ProductService(product_pb2_grpc.ProductServiceServicer):
    def GetProduct(self, request, context):
        cache_key = f"grpc:response:GetProduct:{request.id}"

        # Check cache
        cached = r.get(cache_key)
        if cached:
            # Deserialize from cached protobuf bytes
            response = product_pb2.Product()
            response.ParseFromString(cached)
            context.send_initial_metadata([('x-cache', 'HIT')])
            return response

        # Fetch from database
        product = db.get_product(request.id)
        if not product:
            context.abort(grpc.StatusCode.NOT_FOUND, f"Product {request.id} not found")
            return

        response = product_pb2.Product(
            id=product['id'],
            name=product['name'],
            price=product['price']
        )

        # Cache serialized protobuf bytes
        r.setex(cache_key, 300, response.SerializeToString())
        context.send_initial_metadata([('x-cache', 'MISS')])
        return response
```

## Caching List Responses

```python
def ListProducts(self, request, context):
    # Build a stable cache key from request fields
    request_hash = hashlib.md5(request.SerializeToString()).hexdigest()
    cache_key = f"grpc:response:ListProducts:{request_hash}"

    cached = r.get(cache_key)
    if cached:
        response = product_pb2.ListProductsResponse()
        response.ParseFromString(cached)
        return response

    products = db.list_products(
        category=request.category,
        page=request.page,
        page_size=request.page_size
    )

    response = product_pb2.ListProductsResponse(
        products=[product_pb2.Product(**p) for p in products]
    )

    r.setex(cache_key, 60, response.SerializeToString())
    return response
```

## Cache Invalidation on Writes

```python
def UpdateProduct(self, request, context):
    updated = db.update_product(request.id, request)

    # Invalidate all cached responses for this product
    r.delete(f"grpc:response:GetProduct:{request.id}")

    # Invalidate list caches - they may include this product
    list_keys = r.keys("grpc:response:ListProducts:*")
    if list_keys:
        r.delete(*list_keys)

    return product_pb2.Product(**updated)
```

## Go Client with Response Caching

```go
func (c *CachingProductClient) GetProduct(ctx context.Context, id string) (*pb.Product, error) {
    cacheKey := fmt.Sprintf("grpc:response:GetProduct:%s", id)

    val, err := c.redis.Get(ctx, cacheKey).Bytes()
    if err == nil {
        product := &pb.Product{}
        if err := proto.Unmarshal(val, product); err == nil {
            return product, nil
        }
    }

    product, err := c.client.GetProduct(ctx, &pb.GetProductRequest{Id: id})
    if err != nil {
        return nil, err
    }

    data, _ := proto.Marshal(product)
    c.redis.SetEx(ctx, cacheKey, data, 5*time.Minute)
    return product, nil
}
```

## Summary

Caching gRPC responses in Redis using protobuf serialization is efficient because the binary format is already compact and requires no transformation. Unary RPCs are the best candidates for caching. Use the serialized request as a cache key hash for parameterized queries, and invalidate related list caches when write operations occur. Storing protobuf bytes directly avoids the overhead of JSON marshaling on both cache write and read paths.
