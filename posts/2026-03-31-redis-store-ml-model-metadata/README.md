# How to Store ML Model Metadata in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Machine Learning, Model Registry, Metadata, MLOps

Description: Use Redis to store and retrieve ML model metadata such as versions, metrics, and deployment status, enabling fast model lookups during inference and CI/CD pipelines.

---

ML model registries like MLflow are powerful but heavyweight. For teams that need a lightweight, fast-access store for model versions, evaluation metrics, and deployment status, Redis provides a practical alternative using Hashes, Sorted Sets, and Pub/Sub.

## Model Record Structure

Store each model version as a Hash:

```bash
HSET model:fraud_detector:v2.3.1 \
  model_name "fraud_detector" \
  version "v2.3.1" \
  artifact_path "s3://models/fraud_detector/v2.3.1/model.pkl" \
  training_date "2026-03-15" \
  framework "scikit-learn" \
  precision "0.921" \
  recall "0.884" \
  f1_score "0.902" \
  training_rows "1200000" \
  deployed "false" \
  created_by "alice"
```

## Registering a New Model Version

```python
import redis
import time

r = redis.Redis(host="redis", port=6379, decode_responses=True)

def register_model(
    name: str, version: str, artifact_path: str,
    metrics: dict, metadata: dict
):
    key = f"model:{name}:{version}"

    r.hset(key, mapping={
        "model_name": name,
        "version": version,
        "artifact_path": artifact_path,
        "registered_at": str(int(time.time())),
        "deployed": "false",
        **{k: str(v) for k, v in metrics.items()},
        **{k: str(v) for k, v in metadata.items()}
    })

    # Add to sorted set: score = registration timestamp
    r.zadd(f"model_versions:{name}", {version: time.time()})

    # Publish registration event
    r.publish("model:events", f"registered:{name}:{version}")
    return key
```

## Listing Versions for a Model

```python
def list_versions(model_name: str) -> list:
    # Returns versions sorted by registration time (newest last)
    return r.zrange(f"model_versions:{model_name}", 0, -1)

def get_latest_version(model_name: str) -> str | None:
    versions = r.zrange(f"model_versions:{model_name}", -1, -1)
    return versions[0] if versions else None
```

## Promoting a Model to Production

```python
def promote_to_production(model_name: str, version: str):
    # Demote current production model
    current = r.get(f"model:production:{model_name}")
    if current:
        r.hset(f"model:{model_name}:{current}", "deployed", "false")

    # Promote new version
    r.set(f"model:production:{model_name}", version)
    r.hset(f"model:{model_name}:{version}", "deployed", "true")
    r.hset(f"model:{model_name}:{version}", "deployed_at",
           str(int(time.time())))

    r.publish("model:events", f"promoted:{model_name}:{version}")

def get_production_model(model_name: str) -> dict | None:
    version = r.get(f"model:production:{model_name}")
    if not version:
        return None
    return r.hgetall(f"model:{model_name}:{version}")
```

## Comparing Model Metrics

```python
def compare_versions(model_name: str, v1: str, v2: str) -> dict:
    m1 = r.hgetall(f"model:{model_name}:{v1}")
    m2 = r.hgetall(f"model:{model_name}:{v2}")

    metric_keys = ["precision", "recall", "f1_score"]
    comparison = {}
    for k in metric_keys:
        val1 = float(m1.get(k, 0))
        val2 = float(m2.get(k, 0))
        comparison[k] = {"v1": val1, "v2": val2, "delta": round(val2 - val1, 4)}

    return comparison
```

## Model Loading in Inference Service

```python
import joblib
import boto3

_model_cache = {}

def load_model(model_name: str):
    meta = get_production_model(model_name)
    if not meta:
        raise ValueError(f"No production model for {model_name}")

    version = meta["version"]
    if version not in _model_cache:
        # Download from artifact store
        s3 = boto3.client("s3")
        path = meta["artifact_path"]  # s3://bucket/path/model.pkl
        # ... download and load
        _model_cache[version] = joblib.load("/tmp/model.pkl")

    return _model_cache[version]
```

## Summary

Redis provides a fast, queryable model registry using Hashes for per-version metadata, Sorted Sets for chronological version listing, and plain string keys for production pointers. Pub/Sub notifications let inference services reload models automatically on promotion. This pattern suits teams that want lightweight model tracking without running a full MLflow or SageMaker Model Registry.
