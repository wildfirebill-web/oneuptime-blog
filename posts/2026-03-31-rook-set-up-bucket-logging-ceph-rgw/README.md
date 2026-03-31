# How to Set Up Bucket Logging in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, RGW, Logging, Audit, Object Storage

Description: Configure S3-compatible bucket access logging in Ceph RGW to track object requests for auditing, compliance, and debugging purposes.

---

## Why Enable Bucket Logging?

Bucket access logging records every request made against your Ceph RGW bucket, including the requester, object key, response code, and bytes transferred. This is critical for compliance auditing, troubleshooting access issues, and understanding usage patterns.

## Prerequisites

- Ceph RGW running with S3 API access
- AWS CLI or boto3 configured against your RGW endpoint
- A source bucket and a target (log) bucket

## Step 1: Create a Target Log Bucket

Logs are written as objects to a separate bucket. Create it first:

```bash
aws s3 mb s3://my-bucket-logs \
  --endpoint-url http://rgw.example.com:7480
```

## Step 2: Grant Log Delivery Permissions

RGW needs permission to write to the log bucket. Apply a bucket ACL:

```bash
aws s3api put-bucket-acl \
  --bucket my-bucket-logs \
  --grant-write 'URI="http://acs.amazonaws.com/groups/s3/LogDelivery"' \
  --grant-read-acp 'URI="http://acs.amazonaws.com/groups/s3/LogDelivery"' \
  --endpoint-url http://rgw.example.com:7480
```

## Step 3: Enable Logging on the Source Bucket

Create a logging configuration JSON:

```json
{
  "LoggingEnabled": {
    "TargetBucket": "my-bucket-logs",
    "TargetPrefix": "my-bucket/"
  }
}
```

Apply it:

```bash
aws s3api put-bucket-logging \
  --bucket my-bucket \
  --bucket-logging-status file://logging.json \
  --endpoint-url http://rgw.example.com:7480
```

## Step 4: Verify the Configuration

```bash
aws s3api get-bucket-logging \
  --bucket my-bucket \
  --endpoint-url http://rgw.example.com:7480
```

## Step 5: Generate and View Logs

Make a few requests against the source bucket, then list the log objects:

```bash
aws s3 ls s3://my-bucket-logs/my-bucket/ \
  --endpoint-url http://rgw.example.com:7480
```

Download and inspect a log file:

```bash
aws s3 cp s3://my-bucket-logs/my-bucket/2026-03-31-12-00-00-EXAMPLE \
  ./access.log \
  --endpoint-url http://rgw.example.com:7480

cat access.log
```

Log entries are space-delimited and include fields such as bucket owner, bucket name, timestamp, remote IP, requester, operation, and HTTP status.

## Tuning Log Delivery

RGW delivers logs asynchronously. There may be a delay of minutes before log objects appear. To reduce the log flush interval, adjust:

```ini
[client.rgw.myzone]
rgw_log_object_name = %Y-%m-%d-%H-%M-%S
rgw_usage_log_flush_threshold = 1024
```

## Summary

Ceph RGW supports S3-compatible bucket access logging where request records are stored as objects in a designated log bucket. Enable logging via `put-bucket-logging`, grant write permissions to the log delivery group, and retrieve log files from the target bucket. Logs are delivered asynchronously and are useful for audit and debugging workflows.
