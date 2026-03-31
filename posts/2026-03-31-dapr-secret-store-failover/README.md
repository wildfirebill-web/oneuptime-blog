# How to Configure Secret Store Failover in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Secret, Failover, Resilience, High Availability

Description: Learn how to configure secret store failover in Dapr using fallback chains, application-level caching, and multi-region setups to ensure secrets remain available during outages.

---

## Why Secret Store Failover Matters

A secret store outage can prevent your application from starting or functioning. Dapr does not have a built-in secret store failover mechanism, but you can implement resilient secret access through application-level caching, retry logic, and multi-provider strategies.

## Strategy 1: Application-Level Secret Caching

Cache secrets in memory at startup so the application continues to function if the secret store becomes temporarily unavailable:

```go
package secrets

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"

    dapr "github.com/dapr/go-sdk/client"
)

type CachedSecretStore struct {
    client     dapr.Client
    storeName  string
    cache      map[string]cachedSecret
    mu         sync.RWMutex
    cacheTTL   time.Duration
}

type cachedSecret struct {
    value     map[string]string
    expiresAt time.Time
}

func NewCachedSecretStore(client dapr.Client, storeName string, ttl time.Duration) *CachedSecretStore {
    return &CachedSecretStore{
        client:    client,
        storeName: storeName,
        cache:     make(map[string]cachedSecret),
        cacheTTL:  ttl,
    }
}

func (s *CachedSecretStore) GetSecret(ctx context.Context, key string) (map[string]string, error) {
    // Check cache first
    s.mu.RLock()
    if cached, ok := s.cache[key]; ok && time.Now().Before(cached.expiresAt) {
        s.mu.RUnlock()
        return cached.value, nil
    }
    s.mu.RUnlock()

    // Try to fetch from Dapr secret store
    secret, err := s.client.GetSecret(ctx, s.storeName, key, nil)
    if err != nil {
        // Return cached value even if expired, as fallback
        s.mu.RLock()
        if cached, ok := s.cache[key]; ok {
            log.Printf("Secret store unavailable, using stale cache for %s", key)
            s.mu.RUnlock()
            return cached.value, nil
        }
        s.mu.RUnlock()
        return nil, fmt.Errorf("secret store unavailable and no cache for %s: %w", key, err)
    }

    // Update cache
    s.mu.Lock()
    s.cache[key] = cachedSecret{
        value:     secret,
        expiresAt: time.Now().Add(s.cacheTTL),
    }
    s.mu.Unlock()

    return secret, nil
}
```

## Strategy 2: Multiple Secret Store Components

Define multiple secret store components pointing to different backends (e.g., primary and fallback):

```yaml
# Primary: HashiCorp Vault
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: primary-secrets
  namespace: default
spec:
  type: secretstores.hashicorp.vault
  version: v1
  metadata:
  - name: vaultAddr
    value: "https://vault-primary.example.com:8200"
  - name: vaultTokenMountPath
    value: "/var/run/secrets/vault/token"
---
# Fallback: Kubernetes Secrets
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: fallback-secrets
  namespace: default
spec:
  type: secretstores.kubernetes
  version: v1
```

## Strategy 3: Fallback Chain in Application Code

```python
from dapr.clients import DaprClient
from typing import Optional

class FailoverSecretClient:
    """Tries primary store first, falls back to secondary on error."""

    def __init__(self, primary_store: str, fallback_store: str):
        self.primary = primary_store
        self.fallback = fallback_store

    def get_secret(self, key: str) -> Optional[dict]:
        with DaprClient() as client:
            # Try primary store
            try:
                secret = client.get_secret(store_name=self.primary, key=key)
                return secret.secret
            except Exception as primary_err:
                print(f"Primary secret store failed for '{key}': {primary_err}")

            # Fall back to secondary store
            try:
                secret = client.get_secret(store_name=self.fallback, key=key)
                print(f"Using fallback store for secret '{key}'")
                return secret.secret
            except Exception as fallback_err:
                print(f"Fallback secret store also failed for '{key}': {fallback_err}")
                return None

# Usage
store = FailoverSecretClient("primary-secrets", "fallback-secrets")
creds = store.get_secret("database/credentials")
```

## Strategy 4: Pre-Loading Secrets at Startup

Load all required secrets at startup before serving traffic:

```javascript
const { DaprClient } = require('@dapr/dapr');

const secretCache = new Map();
const client = new DaprClient();

const REQUIRED_SECRETS = [
  'database/password',
  'api/stripe-key',
  'jwt/signing-key',
];

async function preloadSecrets() {
  console.log('Preloading secrets...');

  for (const key of REQUIRED_SECRETS) {
    try {
      const secret = await client.secret.get('primary-secrets', key);
      secretCache.set(key, secret);
      console.log(`Loaded: ${key}`);
    } catch (err) {
      console.error(`Failed to preload ${key}: ${err.message}`);
      // Startup fails if critical secrets are unavailable
      throw new Error(`Required secret unavailable: ${key}`);
    }
  }
  console.log('All secrets preloaded');
}

// Start server only after secrets are loaded
await preloadSecrets();
app.listen(3000);
```

## Summary

Dapr does not natively support secret store failover, but you can implement it through application-level caching with stale fallback, a failover chain that tries multiple stores in sequence, and pre-loading secrets at startup. For production systems, combine these approaches: preload critical secrets at startup, cache them with a TTL, and fall back to stale cache or a secondary store when the primary is unavailable.
