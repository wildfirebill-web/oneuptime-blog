# How to Use S3 Select to Query Data in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, RGW, S3 Select, Query, Analytics

Description: Use S3 Select with Ceph RGW to run SQL-like queries on CSV and JSON objects in place, retrieving only the relevant data without downloading entire files.

---

## Overview

S3 Select allows you to run SQL expressions against the contents of a stored object, returning only the matching rows rather than the entire file. Ceph RGW supports S3 Select for CSV and JSON objects, making it useful for log analysis, data filtering, and lightweight analytics directly on stored data.

## Enable S3 Select in Ceph RGW

S3 Select requires the Ceph Lua scripting engine and the select feature flag. Verify the feature is enabled:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config get client.rgw rgw_enable_static_website
```

Enable S3 Select support:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_s3select_default_prefetch_size 65536
```

## Upload a CSV File

Create a sample CSV:

```bash
cat << 'EOF' > /tmp/sales.csv
id,product,quantity,price,region
1,Widget A,100,9.99,us-east
2,Widget B,50,19.99,us-west
3,Widget C,200,4.99,eu-west
4,Widget A,75,9.99,eu-east
5,Widget B,30,19.99,us-east
EOF

aws s3 cp /tmp/sales.csv s3://my-bucket/data/sales.csv \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Run an S3 Select Query via AWS CLI

Query rows where region is us-east:

```bash
aws s3api select-object-content \
  --bucket my-bucket \
  --key data/sales.csv \
  --expression "SELECT * FROM S3Object s WHERE s.region = 'us-east'" \
  --expression-type SQL \
  --input-serialization '{"CSV": {"FileHeaderInfo": "USE"}}' \
  --output-serialization '{"CSV": {}}' \
  /tmp/query-results.csv \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph

cat /tmp/query-results.csv
```

## S3 Select with boto3

```python
import boto3
from botocore.client import Config

s3 = boto3.client(
    "s3",
    endpoint_url="http://rook-ceph-rgw-my-store.rook-ceph:80",
    aws_access_key_id="myaccesskey",
    aws_secret_access_key="mysecretkey",
    config=Config(s3={"addressing_style": "path"}),
)

response = s3.select_object_content(
    Bucket="my-bucket",
    Key="data/sales.csv",
    ExpressionType="SQL",
    Expression="SELECT s.product, SUM(CAST(s.quantity AS INT)) FROM S3Object s GROUP BY s.product",
    InputSerialization={"CSV": {"FileHeaderInfo": "USE"}},
    OutputSerialization={"CSV": {}},
)

for event in response["Payload"]:
    if "Records" in event:
        print(event["Records"]["Payload"].decode("utf-8"))
```

## Query JSON Objects

Upload a JSON lines file:

```bash
cat << 'EOF' > /tmp/events.json
{"ts": "2026-03-31T10:00:00", "level": "ERROR", "msg": "Connection timeout"}
{"ts": "2026-03-31T10:01:00", "level": "INFO",  "msg": "Request processed"}
{"ts": "2026-03-31T10:02:00", "level": "ERROR", "msg": "Disk full"}
EOF

aws s3 cp /tmp/events.json s3://my-bucket/logs/events.json \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

Query only errors:

```bash
aws s3api select-object-content \
  --bucket my-bucket \
  --key logs/events.json \
  --expression "SELECT s.ts, s.msg FROM S3Object s WHERE s.level = 'ERROR'" \
  --expression-type SQL \
  --input-serialization '{"JSON": {"Type": "LINES"}}' \
  --output-serialization '{"JSON": {"RecordDelimiter": "\n"}}' \
  /tmp/errors.json \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Summary

S3 Select in Ceph RGW lets you push query logic to the storage layer, reducing the data transferred over the network and simplifying analysis pipelines. It works with both CSV and JSON formats using standard SQL expressions, making it easy to filter and aggregate stored data without downloading entire objects.
