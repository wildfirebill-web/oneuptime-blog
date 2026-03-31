# How to Store Training Data Statistics in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Machine Learning, Statistics, Data Quality, MLOps

Description: Store and query training dataset statistics in Redis for data drift detection, feature validation, and model health monitoring without reprocessing raw data.

---

Training data statistics - means, standard deviations, value distributions, and missing rates - are essential for data drift detection and input validation at inference time. Redis provides a fast store for these reference statistics that any service can query without running a full data pipeline.

## Statistics to Store

For each feature in your training dataset, store:

```text
- min, max, mean, std_dev
- p25, p50, p75, p95, p99
- null_rate
- value_count (for categoricals)
```

## Writing Training Statistics

```python
import redis
import numpy as np
import json

r = redis.Redis(host="redis", port=6379, decode_responses=True)

def store_feature_stats(
    dataset_name: str, version: str,
    feature_name: str, values: np.ndarray
):
    non_null = values[~np.isnan(values)]
    null_rate = (len(values) - len(non_null)) / len(values)
    key = f"stats:{dataset_name}:{version}:{feature_name}"

    r.hset(key, mapping={
        "min": str(float(non_null.min())),
        "max": str(float(non_null.max())),
        "mean": str(float(non_null.mean())),
        "std_dev": str(float(non_null.std())),
        "p25": str(float(np.percentile(non_null, 25))),
        "p50": str(float(np.percentile(non_null, 50))),
        "p75": str(float(np.percentile(non_null, 75))),
        "p95": str(float(np.percentile(non_null, 95))),
        "p99": str(float(np.percentile(non_null, 99))),
        "null_rate": str(round(null_rate, 4)),
        "count": str(len(values))
    })

    # Register the feature in the dataset index
    r.sadd(f"stats_features:{dataset_name}:{version}", feature_name)
```

## Storing Categorical Value Distributions

```python
def store_categorical_stats(
    dataset_name: str, version: str,
    feature_name: str, values: list
):
    from collections import Counter
    counts = Counter(values)
    total = len(values)

    key = f"stats:{dataset_name}:{version}:{feature_name}"
    r.hset(key, mapping={
        "type": "categorical",
        "unique_values": str(len(counts)),
        "count": str(total),
        "top_values": json.dumps(
            {v: c/total for v, c in counts.most_common(20)}
        )
    })
    r.sadd(f"stats_features:{dataset_name}:{version}", feature_name)
```

## Reading Statistics for Drift Detection

```python
def get_feature_stats(dataset_name: str, version: str, feature_name: str) -> dict:
    key = f"stats:{dataset_name}:{version}:{feature_name}"
    raw = r.hgetall(key)
    if not raw:
        return {}

    # Convert numeric strings back to floats
    numeric_keys = ["min", "max", "mean", "std_dev",
                    "p25", "p50", "p75", "p95", "p99", "null_rate"]
    return {k: float(v) if k in numeric_keys else v for k, v in raw.items()}
```

## Drift Detection at Inference Time

```python
def check_feature_drift(
    dataset_name: str, version: str,
    feature_name: str, live_value: float
) -> dict:
    stats = get_feature_stats(dataset_name, version, feature_name)
    if not stats:
        return {"status": "no_baseline"}

    mean = stats["mean"]
    std = stats["std_dev"]
    z_score = (live_value - mean) / std if std > 0 else 0

    return {
        "feature": feature_name,
        "value": live_value,
        "z_score": round(z_score, 3),
        "drift": abs(z_score) > 3,
        "training_mean": mean,
        "training_std": std
    }
```

## Comparing Dataset Versions

```python
def compare_versions(
    dataset_name: str, v1: str, v2: str,
    feature_name: str
) -> dict:
    s1 = get_feature_stats(dataset_name, v1, feature_name)
    s2 = get_feature_stats(dataset_name, v2, feature_name)

    return {
        "feature": feature_name,
        "mean_delta": round(s2["mean"] - s1["mean"], 4),
        "std_delta": round(s2["std_dev"] - s1["std_dev"], 4),
        "null_rate_delta": round(s2["null_rate"] - s1["null_rate"], 4)
    }
```

## Listing All Features in a Dataset

```bash
SMEMBERS stats_features:user_activity:v3.0
# age, account_age_days, total_orders, avg_order_value, ...
```

## Summary

Storing training data statistics in Redis as Hash entries per feature enables fast drift detection at inference time via z-score checks without re-querying the training set. Categorical distributions stored as JSON within a Hash field support value novelty detection. Tracking multiple dataset versions lets you compare distributions and identify data quality regressions before they impact model performance.
