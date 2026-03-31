# How to Configure Bucket CORS in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, RGW, CORS, Object Storage, Web

Description: Configure Cross-Origin Resource Sharing (CORS) rules on Ceph RGW buckets to allow browser-based applications to access objects directly.

---

## Why Configure CORS?

When a browser-based application accesses an RGW bucket directly (e.g., for direct uploads or streaming), the browser enforces the Same-Origin Policy. CORS rules on the bucket tell RGW to include the appropriate headers in responses so browsers allow cross-origin requests.

## Prerequisites

- Ceph RGW cluster with S3 API access
- AWS CLI or boto3 pointed at your RGW endpoint

## Creating a CORS Configuration

Define allowed origins, methods, and headers in a JSON file:

```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["https://app.example.com"],
      "AllowedMethods": ["GET", "PUT", "POST", "DELETE"],
      "AllowedHeaders": ["*"],
      "ExposeHeaders": ["ETag", "x-amz-request-id"],
      "MaxAgeSeconds": 3600
    }
  ]
}
```

For development, you can use a wildcard origin (not recommended for production):

```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["*"],
      "AllowedMethods": ["GET"],
      "AllowedHeaders": ["Authorization"]
    }
  ]
}
```

## Applying the CORS Policy

```bash
aws s3api put-bucket-cors \
  --bucket my-public-bucket \
  --cors-configuration file://cors.json \
  --endpoint-url http://rgw.example.com:7480
```

## Verifying the Policy

```bash
aws s3api get-bucket-cors \
  --bucket my-public-bucket \
  --endpoint-url http://rgw.example.com:7480
```

## Testing with a Preflight Request

Simulate a browser preflight OPTIONS request using curl:

```bash
curl -v -X OPTIONS \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: PUT" \
  -H "Access-Control-Request-Headers: Content-Type" \
  http://rgw.example.com:7480/my-public-bucket/
```

The response should include:

```yaml
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, PUT, POST, DELETE
Access-Control-Max-Age: 3600
```

## Removing CORS Rules

To delete the CORS configuration from a bucket:

```bash
aws s3api delete-bucket-cors \
  --bucket my-public-bucket \
  --endpoint-url http://rgw.example.com:7480
```

## Common Issues

- **CORS rules not applying** - Ensure the origin in your request exactly matches an `AllowedOrigins` entry (case-sensitive).
- **PUT blocked despite CORS** - Check that `PUT` is listed in `AllowedMethods` and the bucket policy allows the operation.
- **Preflight fails** - Some clients send a `Host` header that must match the RGW endpoint configuration.

## Summary

Ceph RGW supports S3-compatible CORS configurations applied per bucket. Use `put-bucket-cors` to allow specific origins and HTTP methods, test with an OPTIONS preflight request using curl, and remove policies with `delete-bucket-cors`. Always restrict `AllowedOrigins` to known trusted domains in production.
