# How to Serialize Python Objects for Redis Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Python, Serialization

Description: Learn how to serialize Python objects for Redis storage using pickle, JSON, and MessagePack, with best practices for type safety and schema evolution.

---

Redis stores bytes. To store Python objects, you need serialization. The right choice depends on your needs around performance, readability, type fidelity, and cross-language compatibility.

## Option 1: Pickle (Python-Only, Full Fidelity)

```python
import pickle
import redis

client = redis.Redis(host="localhost", port=6379)

class UserProfile:
    def __init__(self, user_id: int, name: str, score: float):
        self.user_id = user_id
        self.name = name
        self.score = score

profile = UserProfile(user_id=42, name="Alice", score=98.5)

# Serialize and store
client.set("user:42", pickle.dumps(profile))

# Retrieve and deserialize
raw = client.get("user:42")
loaded = pickle.loads(raw)
print(loaded.name)  # Alice
```

Pickle supports arbitrary Python objects but is Python-specific. Never unpickle data from untrusted sources.

## Option 2: JSON (Human-Readable, Cross-Language)

```python
import json
import redis
from dataclasses import dataclass, asdict

@dataclass
class UserProfile:
    user_id: int
    name: str
    score: float

client = redis.Redis(host="localhost", port=6379)
profile = UserProfile(user_id=42, name="Alice", score=98.5)

# Serialize
client.set("user:42", json.dumps(asdict(profile)))

# Deserialize
raw = client.get("user:42")
data = json.loads(raw)
loaded = UserProfile(**data)
```

## Option 3: MessagePack (Compact Binary, Cross-Language)

```python
import msgpack
import redis

client = redis.Redis(host="localhost", port=6379)

data = {"user_id": 42, "name": "Alice", "score": 98.5, "tags": ["premium", "active"]}

# Pack is ~30-50% smaller than JSON
packed = msgpack.packb(data, use_bin_type=True)
client.set("user:42", packed)

# Unpack
raw = client.get("user:42")
unpacked = msgpack.unpackb(raw, raw=False)
```

## Build a Generic Serializer

```python
from enum import Enum
from typing import Any

class SerializationFormat(Enum):
    JSON = "json"
    MSGPACK = "msgpack"
    PICKLE = "pickle"

def serialize(obj: Any, fmt: SerializationFormat = SerializationFormat.JSON) -> bytes:
    if fmt == SerializationFormat.JSON:
        return json.dumps(obj).encode("utf-8")
    elif fmt == SerializationFormat.MSGPACK:
        return msgpack.packb(obj, use_bin_type=True)
    elif fmt == SerializationFormat.PICKLE:
        return pickle.dumps(obj)

def deserialize(data: bytes, fmt: SerializationFormat = SerializationFormat.JSON) -> Any:
    if fmt == SerializationFormat.JSON:
        return json.loads(data)
    elif fmt == SerializationFormat.MSGPACK:
        return msgpack.unpackb(data, raw=False)
    elif fmt == SerializationFormat.PICKLE:
        return pickle.loads(data)
```

## Custom Encoder for datetime and UUID

```python
import json
from datetime import datetime
from uuid import UUID

class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return {"__type__": "datetime", "value": obj.isoformat()}
        if isinstance(obj, UUID):
            return {"__type__": "uuid", "value": str(obj)}
        return super().default(obj)

data = {"created_at": datetime.now(), "id": UUID("12345678-1234-5678-1234-567812345678")}
client.set("event:1", json.dumps(data, cls=CustomEncoder))
```

## Summary

Choose serialization format based on your requirements: pickle for full Python type fidelity in single-service scenarios, JSON for human readability and cross-language compatibility, and MessagePack for compact binary encoding with broad language support. Always version your serialized schemas to handle backward compatibility as your data models evolve.
