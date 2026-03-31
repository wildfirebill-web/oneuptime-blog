# How to Use Redis for Trace Sampling Decisions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Tracing, OpenTelemetry, Observability, Sampling

Description: Use Redis to store and coordinate trace sampling decisions across distributed services, enabling consistent head and tail sampling.

---

Distributed tracing generates enormous volumes of data. Sampling reduces storage costs, but it must be consistent - if service A decides to sample a trace, all downstream services must agree. Redis provides the shared state needed to coordinate these decisions across your fleet.

## The Sampling Decision Problem

Without shared state, each service makes an independent sampling decision. This fragments traces: you get spans from some services but not others, making the trace useless for debugging. Redis solves this by storing the sampling decision keyed on trace ID.

## Head-Based Sampling with Redis

In head-based sampling, the root service makes the decision and stores it. All downstream services look it up:

```python
import redis
import random

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

SAMPLE_RATE = 0.1  # 10% of traces

def get_or_create_sampling_decision(trace_id: str) -> bool:
    key = f"sample:{trace_id}"
    cached = r.get(key)
    if cached is not None:
        return cached == "1"

    # Root service decides
    should_sample = random.random() < SAMPLE_RATE
    value = "1" if should_sample else "0"
    # TTL slightly longer than max trace duration
    r.setex(key, 600, value)
    return should_sample

def should_sample_span(trace_id: str, is_root: bool = False) -> bool:
    if is_root:
        return get_or_create_sampling_decision(trace_id)
    key = f"sample:{trace_id}"
    decision = r.get(key)
    if decision is None:
        # Parent hasn't registered yet - default to sample
        r.setex(key, 600, "1")
        return True
    return decision == "1"
```

## Priority Sampling for Errors

Always sample traces that contain errors, regardless of rate:

```python
def mark_trace_as_error(trace_id: str):
    key = f"sample:{trace_id}"
    r.setex(key, 600, "1")  # force sample

def mark_trace_as_slow(trace_id: str, duration_ms: float, threshold_ms: float = 1000):
    if duration_ms > threshold_ms:
        key = f"sample:{trace_id}"
        r.setex(key, 600, "1")  # force sample slow traces
```

## Tail-Based Sampling

Tail sampling buffers spans and decides after seeing the full trace. Use a Redis hash to accumulate span metadata:

```python
import json

def buffer_span(trace_id: str, span: dict):
    key = f"trace_buffer:{trace_id}"
    r.hset(key, span["span_id"], json.dumps(span))
    r.expire(key, 120)  # 2-minute buffer

def evaluate_and_flush(trace_id: str) -> list:
    key = f"trace_buffer:{trace_id}"
    spans_raw = r.hgetall(key)
    spans = [json.loads(v) for v in spans_raw.values()]

    has_error = any(s.get("status") == "error" for s in spans)
    max_duration = max((s.get("duration_ms", 0) for s in spans), default=0)
    should_keep = has_error or max_duration > 500 or random.random() < 0.05

    if should_keep:
        r.delete(key)
        return spans

    r.delete(key)
    return []
```

## Tracking Sampling Rates

Monitor actual sampling rates using a Redis counter:

```bash
# Increment sampled / total counters
redis-cli incr sampling:sampled:minute:$(date +%H%M)
redis-cli incr sampling:total:minute:$(date +%H%M)

# Get current rate
redis-cli mget sampling:sampled:minute:$(date +%H%M) sampling:total:minute:$(date +%H%M)
```

## Summary

Redis provides a lightweight shared store for coordinating trace sampling decisions across distributed services. Head-based sampling stores decisions at trace start for downstream services to read, while tail-based sampling buffers spans for smarter keep/drop logic. Priority rules ensure errors and slow traces are never dropped.
