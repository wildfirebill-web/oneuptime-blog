# How to Configure Bucket Logging in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Logging, Audit, Object Storage, Compliance, S3

Description: Configure bucket access logging in Ceph RGW to record all S3 API requests to a target bucket for auditing, compliance, and troubleshooting.

---

Ceph RGW bucket logging records detailed information about every S3 API request made to a source bucket, writing log entries to a designated target bucket. This is compatible with the AWS S3 server access logging API and is useful for auditing, compliance, and debugging.

## Enabling Bucket Logging

Bucket logging requires two buckets: a source bucket (the one being logged) and a target bucket (where logs are written). These can be the same bucket or different ones.

Create the target log bucket:

```bash
aws s3api create-bucket \
  --bucket access-logs \
  --endpoint-url http://your-rgw-host:7480
```

## Configuring Logging on the Source Bucket

Enable logging on your source bucket:

```bash
aws s3api put-bucket-logging \
  --bucket mybucket \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "access-logs",
      "TargetPrefix": "mybucket-logs/"
    }
  }' \
  --endpoint-url http://your-rgw-host:7480
```

The `TargetPrefix` adds a prefix to all log object keys, making it easier to organize logs from multiple source buckets.

## Verifying Logging Configuration

```bash
aws s3api get-bucket-logging \
  --bucket mybucket \
  --endpoint-url http://your-rgw-host:7480
```

## Log Object Format

After requests are made to the source bucket, log files appear in the target bucket:

```bash
aws s3 ls s3://access-logs/mybucket-logs/ \
  --endpoint-url http://your-rgw-host:7480
```

Each log file contains lines like:

```
79a59df900b949e55d96a1e698fbacedfd6e09d98eacf8f8d5218e7cd47ef2be mybucket [10/Jan/2026:09:00:00 +0000] 192.168.1.100 alice REQID REST.GET.OBJECT key.txt "GET /mybucket/key.txt HTTP/1.1" 200 - 2048 2048 30 20 "-" "aws-cli/2.0" -
```

Fields include: bucket owner, bucket, time, IP, requester, request ID, operation, key, request URI, HTTP status, error code, bytes sent, object size, total time, turn-around time, referrer, user agent, version ID.

## Enabling Ops Log for Real-Time Logging

For real-time operational logging (not S3 access log format), enable the RGW ops log:

```bash
ceph config set client.rgw rgw_enable_ops_log true
ceph config set client.rgw rgw_ops_log_rados true
```

Query ops log entries:

```bash
radosgw-admin log list
radosgw-admin log show --object <log-object>
```

## Disabling Bucket Logging

```bash
aws s3api put-bucket-logging \
  --bucket mybucket \
  --bucket-logging-status '{}' \
  --endpoint-url http://your-rgw-host:7480
```

## Log Retention with Lifecycle Rules

Automatically expire old logs:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket access-logs \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "expire-old-logs",
      "Status": "Enabled",
      "Filter": {"Prefix": "mybucket-logs/"},
      "Expiration": {"Days": 90}
    }]
  }' \
  --endpoint-url http://your-rgw-host:7480
```

## Summary

Ceph RGW bucket logging provides AWS-compatible S3 access logs written to a target bucket. Enable it per-bucket with a target bucket and prefix, and combine with lifecycle rules to manage log retention. For real-time event streams, use the RGW ops log or bucket notifications instead.
