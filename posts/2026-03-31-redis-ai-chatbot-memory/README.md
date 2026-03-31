# How to Build an AI Chatbot Memory with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, AI, Chatbot, Memory

Description: Give your AI chatbot persistent, context-aware memory using Redis to store conversation history and retrieve relevant past exchanges.

---

AI chatbots without memory forget everything between turns - or even between sessions. Redis provides a simple, fast store for both short-term conversation context and long-term episodic memory using lists and vector search.

## Installation

```bash
pip install redis openai sentence-transformers numpy
```

## Short-Term Memory: Conversation History

Keep recent turns in a Redis list, trimmed to a window size:

```python
import redis
import json
import openai

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
client = openai.OpenAI()

WINDOW_SIZE = 10  # keep last 10 turns

def add_message(session_id: str, role: str, content: str):
    key = f"chat:history:{session_id}"
    message = json.dumps({"role": role, "content": content})
    r.rpush(key, message)
    r.ltrim(key, -WINDOW_SIZE * 2, -1)  # keep last N turns (each turn = 2 messages)
    r.expire(key, 3600)  # expire after 1 hour of inactivity

def get_history(session_id: str) -> list:
    key = f"chat:history:{session_id}"
    raw = r.lrange(key, 0, -1)
    return [json.loads(m) for m in raw]

def chat(session_id: str, user_message: str) -> str:
    add_message(session_id, "user", user_message)
    history = get_history(session_id)

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "You are a helpful assistant."}
        ] + history
    )
    answer = response.choices[0].message.content
    add_message(session_id, "assistant", answer)
    return answer
```

## Long-Term Memory: Episodic Storage

Store important facts and past summaries as vector embeddings for semantic recall:

```python
import numpy as np
from sentence_transformers import SentenceTransformer

r_bin = redis.Redis(host='localhost', port=6379, decode_responses=False)
embedder = SentenceTransformer('all-MiniLM-L6-v2')

r_bin.execute_command(
    'FT.CREATE', 'memory_idx', 'ON', 'HASH',
    'PREFIX', '1', 'mem:',
    'SCHEMA',
    'user_id', 'TAG',
    'content', 'TEXT',
    'embedding', 'VECTOR', 'HNSW', '6',
    'TYPE', 'FLOAT32', 'DIM', '384',
    'DISTANCE_METRIC', 'COSINE'
)

def store_memory(user_id: str, content: str, memory_id: str):
    vec = embedder.encode(content, normalize_embeddings=True)
    r_bin.hset(f"mem:{memory_id}", mapping={
        "user_id": user_id.encode(),
        "content": content.encode(),
        "embedding": vec.astype(np.float32).tobytes()
    })

def recall_memories(user_id: str, query: str, top_k: int = 3):
    query_vec = embedder.encode(query, normalize_embeddings=True)
    query_bytes = query_vec.astype(np.float32).tobytes()

    search_query = (
        f"(@user_id:{{{user_id}}})"
        f"=>[KNN {top_k} @embedding $vec AS score]"
    )

    result = r_bin.execute_command(
        'FT.SEARCH', 'memory_idx', search_query,
        'PARAMS', 2, 'vec', query_bytes,
        'RETURN', 2, 'content', 'score',
        'SORTBY', 'score',
        'DIALECT', 2
    )

    memories = []
    for i in range(1, len(result), 2):
        raw = result[i+1]
        fields = {}
        for j in range(0, len(raw), 2):
            fields[raw[j].decode()] = raw[j+1].decode()
        memories.append(fields['content'])
    return memories
```

## Memory-Augmented Chat

Inject recalled memories into the system prompt:

```python
def memory_chat(user_id: str, session_id: str,
                user_message: str) -> str:
    memories = recall_memories(user_id, user_message)
    memory_context = ""
    if memories:
        memory_context = "\n\nRelevant past context:\n" + "\n".join(
            f"- {m}" for m in memories
        )

    history = get_history(session_id)
    add_message(session_id, "user", user_message)

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content":
             f"You are a helpful assistant.{memory_context}"}
        ] + history + [{"role": "user", "content": user_message}]
    )
    answer = response.choices[0].message.content
    add_message(session_id, "assistant", answer)
    return answer
```

## Summary

Redis provides two complementary memory layers for AI chatbots: a short-term sliding window of recent conversation turns stored in a list, and long-term episodic memory stored as vector embeddings for semantic recall. This architecture keeps context windows small while allowing the chatbot to remember important facts across sessions. Use TTLs on session keys to automatically clean up inactive conversations.
