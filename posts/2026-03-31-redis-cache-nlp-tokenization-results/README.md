# How to Cache NLP Tokenization Results with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, NLP, Cache

Description: Speed up NLP pipelines by caching tokenization results in Redis, avoiding repeated expensive tokenizer calls on the same input text.

---

Tokenization is one of the most frequent operations in NLP pipelines. Whether you are running BERT, GPT, or spaCy, tokenizing the same text repeatedly wastes CPU cycles. Caching tokenization results in Redis lets you skip that overhead entirely.

## Why Cache Tokenization?

Tokenizers like HuggingFace's `transformers` tokenizer are not cheap when running at scale. If your API receives the same article, product description, or user query repeatedly, you are re-tokenizing identical input. Redis lets you store the result keyed on a hash of the input.

## Setting Up the Cache

Install the dependencies:

```bash
pip install redis transformers torch
```

Connect to Redis and initialize your tokenizer:

```python
import redis
import hashlib
import json
from transformers import AutoTokenizer

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")

CACHE_TTL = 3600  # 1 hour
```

## Writing the Cache Wrapper

```python
def get_cache_key(text: str, model_name: str) -> str:
    payload = f"{model_name}:{text}"
    return "tok:" + hashlib.sha256(payload.encode()).hexdigest()

def tokenize_with_cache(text: str, model_name: str = "bert-base-uncased") -> dict:
    key = get_cache_key(text, model_name)

    cached = r.get(key)
    if cached:
        return json.loads(cached)

    result = tokenizer(
        text,
        return_tensors="pt",
        truncation=True,
        max_length=512,
        padding=True
    )

    # Convert tensors to serializable lists
    serializable = {k: v.tolist() for k, v in result.items()}
    r.setex(key, CACHE_TTL, json.dumps(serializable))

    return serializable
```

## Benchmark the Improvement

```python
import time

sample_text = "The quick brown fox jumps over the lazy dog."

# First call - cache miss
start = time.perf_counter()
result1 = tokenize_with_cache(sample_text)
cold = time.perf_counter() - start

# Second call - cache hit
start = time.perf_counter()
result2 = tokenize_with_cache(sample_text)
warm = time.perf_counter() - start

print(f"Cold: {cold*1000:.2f}ms | Warm: {warm*1000:.2f}ms")
# Example output: Cold: 12.45ms | Warm: 0.38ms
```

## Batch Tokenization with Cache

For pipelines processing large datasets, check the cache in bulk using `mget`:

```python
def batch_tokenize(texts: list[str], model_name: str = "bert-base-uncased") -> list[dict]:
    keys = [get_cache_key(t, model_name) for t in texts]
    cached_values = r.mget(keys)

    results = []
    miss_indices = []
    miss_texts = []

    for i, val in enumerate(cached_values):
        if val:
            results.append(json.loads(val))
        else:
            results.append(None)
            miss_indices.append(i)
            miss_texts.append(texts[i])

    if miss_texts:
        batch_result = tokenizer(
            miss_texts,
            return_tensors="pt",
            truncation=True,
            max_length=512,
            padding=True
        )
        serializable_batch = {k: v.tolist() for k, v in batch_result.items()}

        pipe = r.pipeline()
        for idx, i in enumerate(miss_indices):
            single = {k: v[idx] for k, v in serializable_batch.items()}
            pipe.setex(keys[i], CACHE_TTL, json.dumps(single))
            results[i] = single
        pipe.execute()

    return results
```

## Monitoring Cache Performance

Track hit rates to evaluate effectiveness:

```bash
redis-cli info stats | grep -E "keyspace_hits|keyspace_misses"
```

You can also store counters in Redis itself:

```python
def tokenize_with_metrics(text: str) -> dict:
    key = get_cache_key(text, "bert-base-uncased")
    cached = r.get(key)
    if cached:
        r.incr("tok:hits")
        return json.loads(cached)
    r.incr("tok:misses")
    # ... rest of the function
```

## Key Design Tips

- Use model name in the cache key so switching models does not serve stale results.
- Set a reasonable TTL (1-24 hours) based on how often your input text changes.
- For long documents, consider caching at the sentence level for better reuse.
- Monitor memory usage with `redis-cli info memory` and cap the cache with `maxmemory-policy allkeys-lru`.

## Summary

Caching NLP tokenization results with Redis reduces latency from double-digit milliseconds to sub-millisecond for repeated inputs. The pattern involves hashing input text plus model name as the cache key, serializing tensor outputs to JSON, and using `pipeline()` for batch operations to minimize round trips.
