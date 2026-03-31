# How to Use TS.MGET in Redis to Get Latest from Multiple Time Series

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Time Series, RedisTimeSeries, Commands, NoSql

Description: Learn how to use TS.MGET in Redis to retrieve the latest data point from multiple time series keys in a single command using label filters.

---

## Overview

`TS.MGET` is a RedisTimeSeries command that retrieves the latest sample from multiple time series keys in one operation. Instead of querying each key individually, you can use label-based filters to select a group of time series and fetch their most recent values simultaneously. This is useful for dashboards, monitoring systems, and any use case where you need a current snapshot across many sensors or metrics.

## Prerequisites

RedisTimeSeries must be loaded as a module. You can run it via Docker:

```bash
docker run -p 6379:6379 redis/redis-stack:latest
```

## Creating Sample Time Series

Before using `TS.MGET`, create several time series with labels:

```bash
TS.CREATE sensor:room1 LABELS location room1 type temperature
TS.CREATE sensor:room2 LABELS location room2 type temperature
TS.CREATE sensor:room3 LABELS location room3 type humidity

TS.ADD sensor:room1 * 22.5
TS.ADD sensor:room2 * 21.0
TS.ADD sensor:room3 * 55.3
```

## Basic TS.MGET Usage

Retrieve the latest sample from all time series with the label `type=temperature`:

```bash
TS.MGET FILTER type=temperature
```

Example output:

```text
1) 1) "sensor:room1"
   2) (empty array)
   3) 1) (integer) 1700000001000
      2) "22.5"
2) 1) "sensor:room2"
   2) (empty array)
   3) 1) (integer) 1700000002000
      2) "21.0"
```

## Including Labels in the Response

Use `WITHLABELS` to include all labels in the output:

```bash
TS.MGET WITHLABELS FILTER type=temperature
```

Use `SELECTED_LABELS` for a subset of labels:

```bash
TS.MGET SELECTED_LABELS location FILTER type=temperature
```

## Filtering with Multiple Conditions

You can combine multiple filter conditions:

```bash
TS.MGET FILTER type=temperature location=room1
```

Supported filter operators include:

- `label=value` - exact match
- `label!=value` - not equal
- `label=` - label exists with any value
- `label!=` - label does not exist

```bash
TS.MGET FILTER type=temperature location!=room3
```

## Using TS.MGET in Python

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Get latest sample from all temperature sensors
results = r.execute_command('TS.MGET', 'WITHLABELS', 'FILTER', 'type=temperature')

for key, labels, sample in results:
    timestamp, value = sample
    print(f"{key}: {value} at {timestamp}")
```

## Using TS.MGET in Node.js

```javascript
const { createClient } = require('redis');

async function getLatestTemperatures() {
  const client = createClient();
  await client.connect();

  const results = await client.sendCommand([
    'TS.MGET',
    'WITHLABELS',
    'FILTER',
    'type=temperature'
  ]);

  for (const entry of results) {
    const [key, labels, sample] = entry;
    console.log(`${key}: ${sample[1]} at ${sample[0]}`);
  }

  await client.disconnect();
}

getLatestTemperatures();
```

## Practical Use Case - Dashboard Snapshot

Imagine a factory floor with dozens of sensors. You want to display the current value of all pressure sensors on a dashboard:

```bash
TS.CREATE factory:line1:pressure LABELS line 1 type pressure unit psi
TS.CREATE factory:line2:pressure LABELS line 2 type pressure unit psi
TS.CREATE factory:line3:pressure LABELS line 3 type pressure unit psi

TS.ADD factory:line1:pressure * 102.4
TS.ADD factory:line2:pressure * 98.7
TS.ADD factory:line3:pressure * 105.1

TS.MGET WITHLABELS FILTER type=pressure
```

This returns all three sensors' latest readings in a single round trip.

## Comparison with TS.GET

| Command | Scope | Use Case |
|---------|-------|----------|
| TS.GET | Single key | Fetch latest from one series |
| TS.MGET | Multiple keys via filter | Fetch latest from many series |

## Summary

`TS.MGET` allows you to retrieve the latest data point from multiple time series in a single command by filtering on labels. It is ideal for dashboards and monitoring tools that need a real-time snapshot across many metrics. Use `WITHLABELS` or `SELECTED_LABELS` to include metadata in the response.
