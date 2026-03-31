# How to Set Up Rate Limiting for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Rate Limit, Performance, Object Storage, Throttling, Administration

Description: Configure rate limiting in Ceph RGW to control the number of read and write operations per user and bucket, preventing any single client from monopolizing resources.

---

Ceph RGW rate limiting allows administrators to restrict how many read and write requests users or buckets can make per minute. This prevents individual tenants from monopolizing gateway resources and ensures consistent performance for all users.

## Rate Limit Concepts

RGW rate limits operate on two dimensions:
- **ops**: Maximum number of operations (requests) per minute
- **bytes**: Maximum bytes transferred per minute

Both can be set independently for reads and writes, at the user or bucket level.

## Setting User Rate Limits

Limit a user to 1000 read ops/min and 500 write ops/min:

```bash
radosgw-admin ratelimit set \
  --ratelimit-scope user \
  --uid alice \
  --max-read-ops 1000 \
  --max-write-ops 500
```

Limit byte throughput:

```bash
radosgw-admin ratelimit set \
  --ratelimit-scope user \
  --uid alice \
  --max-read-bytes 104857600 \
  --max-write-bytes 52428800
```

Combined limits:

```bash
radosgw-admin ratelimit set \
  --ratelimit-scope user \
  --uid alice \
  --max-read-ops 1000 \
  --max-read-bytes 104857600 \
  --max-write-ops 500 \
  --max-write-bytes 52428800
```

## Enabling User Rate Limits

Setting the rate limit does not enforce it until enabled:

```bash
radosgw-admin ratelimit enable \
  --ratelimit-scope user \
  --uid alice
```

## Viewing Rate Limit Settings

```bash
radosgw-admin ratelimit get \
  --ratelimit-scope user \
  --uid alice
```

Output:

```json
{
  "enabled": true,
  "max_read_ops": 1000,
  "max_write_ops": 500,
  "max_read_bytes": 104857600,
  "max_write_bytes": 52428800
}
```

## Setting Bucket Rate Limits

```bash
radosgw-admin ratelimit set \
  --ratelimit-scope bucket \
  --bucket shared-bucket \
  --max-read-ops 5000 \
  --max-write-ops 1000

radosgw-admin ratelimit enable \
  --ratelimit-scope bucket \
  --bucket shared-bucket
```

## What Happens When Rate Limit Is Exceeded

When a user or bucket exceeds their rate limit, RGW returns HTTP 503:

```xml
HTTP/1.1 503 SlowDown
<Error>
  <Code>SlowDown</Code>
  <Message>Please reduce your request rate.</Message>
</Error>
```

Handle this in client code with exponential backoff:

```python
import time
import boto3
from botocore.exceptions import ClientError

s3 = boto3.client('s3', endpoint_url='http://your-rgw-host:7480')

def upload_with_retry(bucket, key, data, retries=5):
    delay = 1
    for attempt in range(retries):
        try:
            s3.put_object(Bucket=bucket, Key=key, Body=data)
            return
        except ClientError as e:
            if e.response['Error']['Code'] == 'SlowDown':
                time.sleep(delay)
                delay *= 2
            else:
                raise
    raise Exception("Max retries exceeded")
```

## Disabling Rate Limits

```bash
radosgw-admin ratelimit disable \
  --ratelimit-scope user \
  --uid alice
```

## Summary

Ceph RGW rate limiting controls ops/min and bytes/min at the user and bucket level. Set limits with `ratelimit set`, enable enforcement with `ratelimit enable`, and configure clients to handle 503 SlowDown responses with exponential backoff. Rate limits are essential in multi-tenant deployments to ensure fair resource sharing.
