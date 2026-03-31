# How to Use Protocol Buffers with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Protocol Buffer, Serialization

Description: Learn how to use Protocol Buffers to serialize data for Redis storage, enabling compact binary payloads with strict schema enforcement and backward compatibility.

---

Protocol Buffers (protobuf) offer strict schema enforcement, excellent backward compatibility, and very compact binary encoding. They are ideal when multiple services in different languages share data through Redis and schema consistency is critical.

## Define Your Schema

```protobuf
// user.proto
syntax = "proto3";
package myapp;

message UserProfile {
  int64 user_id = 1;
  string name = 2;
  double score = 3;
  repeated string tags = 4;
  int64 created_at_unix = 5;
}
```

```bash
# Compile for Python
protoc --python_out=. user.proto

# Compile for Java
protoc --java_out=. user.proto
```

## Python with protobuf

```bash
pip install protobuf
```

```python
import redis
from user_pb2 import UserProfile
import time

client = redis.Redis(host="localhost", port=6379)

profile = UserProfile(
    user_id=42,
    name="Alice",
    score=98.5,
    tags=["premium", "active"],
    created_at_unix=int(time.time()),
)

# Serialize to bytes
serialized = profile.SerializeToString()
print(f"Protobuf size: {len(serialized)} bytes")

# Store
client.set("user:42", serialized, ex=3600)

# Retrieve and deserialize
raw = client.get("user:42")
loaded = UserProfile()
loaded.ParseFromString(raw)
print(loaded.name)  # Alice
```

## Java with Spring Boot and protobuf

```java
// pom.xml dependency
// <artifactId>protobuf-java</artifactId>

import com.example.UserProfileProto.UserProfile;

// Build from proto
UserProfile profile = UserProfile.newBuilder()
    .setUserId(42)
    .setName("Alice")
    .setScore(98.5)
    .addTags("premium")
    .setCreatedAtUnix(Instant.now().getEpochSecond())
    .build();

// Serialize and store
byte[] bytes = profile.toByteArray();
redisTemplate.opsForValue().set("user:42", bytes, Duration.ofHours(1));

// Retrieve
byte[] raw = (byte[]) redisTemplate.opsForValue().get("user:42");
UserProfile loaded = UserProfile.parseFrom(raw);
```

## Schema Evolution Best Practices

Protobuf allows safe schema changes:

```protobuf
// v1
message UserProfile {
  int64 user_id = 1;
  string name = 2;
  double score = 3;
}

// v2 - SAFE: added new field, old code ignores field 4
message UserProfile {
  int64 user_id = 1;
  string name = 2;
  double score = 3;
  string email = 4;  // new field - backward compatible
}

// UNSAFE: never remove or renumber existing fields
// SAFE: you can deprecate and stop writing to them
```

## Versioned Key Strategy

```python
# Embed version in key to manage cache invalidation during schema changes
def store_profile(client, profile: UserProfile, schema_version: int = 2):
    key = f"user:{profile.user_id}:v{schema_version}"
    client.set(key, profile.SerializeToString(), ex=3600)

def get_profile(client, user_id: int, schema_version: int = 2):
    key = f"user:{user_id}:v{schema_version}"
    raw = client.get(key)
    if raw is None:
        return None
    profile = UserProfile()
    profile.ParseFromString(raw)
    return profile
```

## Summary

Protocol Buffers provide compact binary encoding and strict schema evolution rules, making them ideal for multi-language Redis data sharing. The `.proto` schema acts as a contract between producers and consumers, and the field numbering system ensures that adding new fields does not break existing readers or writers.
