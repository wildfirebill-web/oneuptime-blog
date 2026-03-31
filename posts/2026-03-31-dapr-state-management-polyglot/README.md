# How to Use Dapr State Management Across Different Language Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, State Management, Polyglot, Redis, Microservice

Description: Share state across Python, Go, Java, and Node.js services using Dapr's state management API with a common Redis state store and consistent key conventions.

---

Dapr state management allows multiple services written in different languages to read and write shared state through a unified API. This is useful for session data, distributed caches, and shared configuration that multiple services need to access.

## Shared State Store Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: shared-state
  namespace: default
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis-master:6379
  - name: actorStateStore
    value: "false"
  - name: keyPrefix
    value: "none"
```

Setting `keyPrefix: none` allows all services to share the same key namespace.

## Python: Reading and Writing State

```python
import requests
import os
import json

DAPR_PORT = os.getenv("DAPR_HTTP_PORT", "3500")
STATE_STORE = "shared-state"

def save_user_session(user_id: str, session_data: dict):
    response = requests.post(
        f"http://localhost:{DAPR_PORT}/v1.0/state/{STATE_STORE}",
        headers={"Content-Type": "application/json"},
        data=json.dumps([{
            "key": f"session:{user_id}",
            "value": session_data
        }])
    )
    return response.status_code == 204

def get_user_session(user_id: str) -> dict:
    response = requests.get(
        f"http://localhost:{DAPR_PORT}/v1.0/state/{STATE_STORE}/session:{user_id}"
    )
    if response.status_code == 200 and response.text:
        return response.json()
    return None
```

## Node.js: Reading State Written by Python

```javascript
const { DaprClient } = require('@dapr/dapr');

const daprClient = new DaprClient();
const STATE_STORE = 'shared-state';

async function getUserSession(userId) {
  const session = await daprClient.state.get(
    STATE_STORE,
    `session:${userId}`
  );
  return session;
}

async function updateSessionActivity(userId) {
  const session = await getUserSession(userId);
  if (session) {
    session.lastActive = new Date().toISOString();
    await daprClient.state.save(STATE_STORE, [
      { key: `session:${userId}`, value: session }
    ]);
  }
}
```

## Go: Transactional State Update

```go
package main

import (
    "context"
    dapr "github.com/dapr/go-sdk/client"
)

func transferCredits(fromUser, toUser string, amount float64) error {
    client, _ := dapr.NewClient()
    defer client.Close()

    ops := []*dapr.StateOperation{
        {
            Type: dapr.StateOperationTypeUpsert,
            Item: &dapr.SetStateItem{
                Key:   "balance:" + fromUser,
                Value: marshalBalance(getBalance(fromUser) - amount),
            },
        },
        {
            Type: dapr.StateOperationTypeUpsert,
            Item: &dapr.SetStateItem{
                Key:   "balance:" + toUser,
                Value: marshalBalance(getBalance(toUser) + amount),
            },
        },
    }

    return client.ExecuteStateTransaction(
        context.Background(),
        "shared-state",
        nil,
        ops,
    )
}
```

## Java: Reading State with ETag Concurrency Control

```java
import io.dapr.client.DaprClient;
import io.dapr.client.DaprClientBuilder;
import io.dapr.client.domain.State;

public class StateService {

    private final DaprClient daprClient = new DaprClientBuilder().build();

    public void updateWithOptimisticLock(String userId, UserProfile profile) {
        State<UserProfile> current = daprClient
            .getState("shared-state", "profile:" + userId, UserProfile.class)
            .block();

        // Use ETag for optimistic concurrency
        daprClient.saveState("shared-state",
            "profile:" + userId,
            current.getEtag(),
            profile,
            null
        ).block();
    }
}
```

## Key Naming Conventions

Use consistent key prefixes to organize shared state across services:

```bash
# Pattern: {entity-type}:{identifier}
session:{user_id}        # User sessions (written by auth service)
balance:{user_id}        # Account balances (managed by payment service)
profile:{user_id}        # User profiles (managed by user service)
cart:{session_id}        # Shopping carts (managed by cart service)
```

## Summary

Dapr state management provides a consistent HTTP API for reading and writing state across Python, Node.js, Go, and Java services. Using a shared component definition with `keyPrefix: none` and a consistent key naming convention enables multiple services to collaborate on shared state while Dapr handles consistency, concurrency control, and the underlying storage technology.
