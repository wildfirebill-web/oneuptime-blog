# How to Use TS.MGET in Redis to Get Latest from Multiple Time Series

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisTimeSeries, Time Series, Multi-Key, Monitoring

Description: Learn how to use TS.MGET in Redis to retrieve the latest sample from multiple time series using label filters in a single command.

---

## What Is TS.MGET?

`TS.MGET` retrieves the most recent sample from all time series that match a set of label filters. Instead of calling `TS.GET` for each key individually, `TS.MGET` lets you fetch the latest value from multiple time series in a single command using label-based selection.

This is ideal for multi-host dashboards, fleet monitoring, and any scenario where you group time series by labels like host, region, or metric type.

## Basic Syntax

```text
TS.MGET [LATEST] [WITHLABELS | SELECTED_LABELS label...]
  FILTER filterExpr [filterExpr ...]
```

Parameters:
- `LATEST` - include data from currently open compaction buckets
- `WITHLABELS` - include all labels in the response
- `SELECTED_LABELS` - include only specific labels in the response
- `FILTER` - one or more label filter expressions

## Setting Up Sample Data

```bash
# Create time series with labels
TS.CREATE cpu:host1 LABELS host host1 region us-east metric cpu
TS.CREATE cpu:host2 LABELS host host2 region us-east metric cpu
TS.CREATE cpu:host3 LABELS host host3 region eu-west metric cpu

TS.CREATE memory:host1 LABELS host host1 region us-east metric memory
TS.CREATE memory:host2 LABELS host host2 region us-east metric memory

# Add samples
TS.ADD cpu:host1 * 72.5
TS.ADD cpu:host2 * 55.3
TS.ADD cpu:host3 * 88.1
TS.ADD memory:host1 * 4096
TS.ADD memory:host2 * 2048
```

## Querying by Label

```bash
# Get latest CPU from all hosts
TS.MGET FILTER metric=cpu

# Get latest metrics from hosts in us-east
TS.MGET FILTER region=us-east

# Get latest CPU from us-east specifically
TS.MGET FILTER metric=cpu region=us-east
```

## Sample Response

```text
1) 1) "cpu:host1"
   2) 1) (empty array)      # labels (empty unless WITHLABELS)
   3) 1) (integer) 1700000000000
      2) "72.5"
2) 1) "cpu:host2"
   2) 1) (empty array)
   3) 1) (integer) 1700000000000
      2) "55.3"
3) 1) "cpu:host3"
   2) 1) (empty array)
   3) 1) (integer) 1700000000000
      2) "88.1"
```

## Including Labels in Response

```bash
# Include all labels in response
TS.MGET WITHLABELS FILTER metric=cpu
```

```text
1) 1) "cpu:host1"
   2) 1) 1) "host"
         2) "host1"
      2) 1) "region"
         2) "us-east"
      3) 1) "metric"
         2) "cpu"
   3) 1) (integer) 1700000000000
      2) "72.5"
```

## Selective Labels

```bash
# Only include the 'host' label to save bandwidth
TS.MGET SELECTED_LABELS host FILTER metric=cpu
```

## Filter Expressions

| Expression | Meaning |
|---|---|
| `label=value` | Label equals value |
| `label!=value` | Label does not equal value |
| `label=` | Label exists but has no value |
| `label!=` | Label does not exist |
| `label=(v1,v2)` | Label equals v1 or v2 |
| `label!=(v1,v2)` | Label is not v1 or v2 |

```bash
# Get CPU from host1 OR host2
TS.MGET FILTER metric=cpu host=(host1,host2)

# Get all metrics except CPU from us-east
TS.MGET FILTER region=us-east metric!=cpu
```

## Python Example

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
ts = r.ts()

# Get latest CPU values from all hosts
results = ts.mget(filters=['metric=cpu'], with_labels=True)

print("Current CPU usage:")
for key, labels, (timestamp, value) in results:
    host = labels.get('host', 'unknown')
    region = labels.get('region', 'unknown')
    print(f"  {host} ({region}): {float(value):.1f}%")
```

## Building a Fleet Dashboard

```python
import redis
from collections import defaultdict

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
ts = r.ts()

def get_fleet_status(region=None):
    """Get latest metrics for all hosts in a region."""
    filters = ['metric=(cpu,memory)']
    if region:
        filters.append(f'region={region}')

    results = ts.mget(filters=filters, with_labels=True)

    fleet = defaultdict(dict)
    for key, labels, sample in results:
        host = labels.get('host', key)
        metric = labels.get('metric', 'unknown')
        timestamp, value = sample
        fleet[host][metric] = float(value)

    return dict(fleet)

fleet = get_fleet_status(region='us-east')
for host, metrics in sorted(fleet.items()):
    cpu = metrics.get('cpu', 'N/A')
    mem = metrics.get('memory', 'N/A')
    print(f"  {host}: CPU={cpu}%, Memory={mem}MB")
```

## Summary

`TS.MGET` is the multi-key variant of `TS.GET`, enabling efficient retrieval of the latest sample from multiple time series using label-based filtering. It eliminates the need for multiple round trips when monitoring metrics across many hosts or services. Use `WITHLABELS` or `SELECTED_LABELS` to include metadata in the response, and combine multiple filter expressions to narrow results to specific subsets of your time series fleet.
