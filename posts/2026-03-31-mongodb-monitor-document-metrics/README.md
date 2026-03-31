# How to Monitor MongoDB Document Metrics (inserted, updated, deleted)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Monitoring, Document, Metrics, Performance

Description: Learn how to monitor MongoDB document-level metrics for inserts, updates, and deletes using metrics.document from serverStatus to track write activity over time.

---

MongoDB's `metrics.document` counters track the actual number of documents affected by write operations, which differs from `opcounters` that count operation calls. A single update targeting 1000 documents counts as 1 in `opcounters.update` but 1000 in `metrics.document.updated`. This distinction is crucial for understanding actual data change volume.

## Reading Document Metrics

```javascript
db.adminCommand({ serverStatus: 1 }).metrics.document
```

Output:

```json
{
  "deleted": 45678,
  "inserted": 1234567,
  "returned": 98765432,
  "updated": 345678
}
```

All counters are cumulative since the last mongod start.

## Key Metrics Explained

```text
inserted  - Total documents inserted across all insert operations
updated   - Total documents matched and modified by update operations
deleted   - Total documents removed by delete operations
returned  - Total documents returned to clients by find operations
```

Compare `metrics.document.updated` with `opcounters.update`:
- If `opcounters.update = 1000` but `metrics.document.updated = 50000`, each update call is modifying 50 documents on average (bulk updates).

## Monitoring Document Write Rates in Python

```python
import pymongo
import time

client = pymongo.MongoClient("mongodb://localhost:27017")

def get_document_metrics():
    status = client.admin.command("serverStatus")
    doc = status["metrics"]["document"]
    return {
        "inserted": doc["inserted"],
        "updated": doc["updated"],
        "deleted": doc["deleted"],
        "returned": doc["returned"],
    }

def get_document_rates(interval=10):
    s1 = get_document_metrics()
    time.sleep(interval)
    s2 = get_document_metrics()

    return {
        "inserted_per_sec": (s2["inserted"] - s1["inserted"]) / interval,
        "updated_per_sec": (s2["updated"] - s1["updated"]) / interval,
        "deleted_per_sec": (s2["deleted"] - s1["deleted"]) / interval,
        "returned_per_sec": (s2["returned"] - s1["returned"]) / interval,
    }

while True:
    rates = get_document_rates(10)
    print(
        f"Docs/s: insert={rates['inserted_per_sec']:.1f}  "
        f"update={rates['updated_per_sec']:.1f}  "
        f"delete={rates['deleted_per_sec']:.1f}  "
        f"returned={rates['returned_per_sec']:.1f}"
    )
```

## Comparing opcounters vs metrics.document

Use both together to diagnose write patterns:

```python
def get_write_analysis():
    status = client.admin.command("serverStatus")
    ops = status["opcounters"]
    docs = status["metrics"]["document"]

    return {
        "update_ops": ops["update"],
        "docs_updated": docs["updated"],
        "avg_docs_per_update": docs["updated"] / max(ops["update"], 1),
        "delete_ops": ops["delete"],
        "docs_deleted": docs["deleted"],
        "avg_docs_per_delete": docs["deleted"] / max(ops["delete"], 1),
    }
```

High `avg_docs_per_update` means your application is doing multi-document updates, which could indicate missing targeted queries or intentional bulk operations.

## Monitoring Returned Documents Ratio

Compare `metrics.document.returned` with `opcounters.query` to measure query efficiency:

```python
def get_read_efficiency():
    status = client.admin.command("serverStatus")
    queries = status["opcounters"]["query"]
    returned = status["metrics"]["document"]["returned"]
    return {
        "queries": queries,
        "docs_returned": returned,
        "avg_docs_per_query": returned / max(queries, 1)
    }
```

A very high `avg_docs_per_query` suggests queries without `limit()` or missing index selectivity.

## Grafana Metrics

Useful Prometheus queries for document metrics:

```text
# Document insert rate
rate(mongodb_metrics_document_inserted_total[5m])

# Ratio of documents returned per query
rate(mongodb_metrics_document_returned_total[5m]) /
rate(mongodb_opcounters_query_total[5m])
```

## Setting Write Volume Alerts

```python
rates = get_document_rates(60)

if rates["deleted_per_sec"] > 1000:
    print(f"ALERT: High delete rate: {rates['deleted_per_sec']:.0f} docs/sec")

if rates["returned_per_sec"] > 100000:
    print(f"ALERT: High document return rate: {rates['returned_per_sec']:.0f} docs/sec - check for missing LIMIT")
```

## Summary

`metrics.document` from `serverStatus` provides document-level granularity beyond what `opcounters` offer. The key insight is that one update operation can affect many documents, so tracking both gives a complete picture of write workload. High `returned` rates relative to `query` counts indicate inefficient read patterns. Monitor these metrics alongside opcounters to distinguish between frequent small operations and infrequent bulk operations that have similar aggregate impact.
