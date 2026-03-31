# How to Handle Actor State Persistence in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actors, State Management, Persistence, Microservices

Description: Learn how Dapr actor state persistence works, how to configure the state store, and best practices for managing actor state reliably in production.

---

## How Dapr Actor State Works

Dapr virtual actors automatically persist their state to the configured state store. Every time an actor writes to its state manager, Dapr batches those changes and commits them atomically at the end of each turn (method call). This means state is durable without explicit save calls in most scenarios.

The state is keyed by actor type, actor ID, and state key, enabling multiple actors of the same type to have independent isolated state.

## Configure the Actor State Store

Define a state store component. For actors, add the `actorStateStore: "true"` metadata:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: default
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "localhost:6379"
  - name: redisPassword
    value: ""
  - name: actorStateStore
    value: "true"
```

Only one state store can be marked as `actorStateStore: "true"` in a namespace.

## Read and Write Actor State (.NET)

```csharp
using Dapr.Actors.Runtime;
using System.Threading.Tasks;

[Actor(TypeName = "AccountActor")]
public class AccountActor : Actor, IAccountActor
{
    private const string BalanceKey = "balance";
    private const string MetadataKey = "metadata";

    public AccountActor(ActorHost host) : base(host) { }

    // Write state
    public async Task SetBalanceAsync(decimal balance)
    {
        await StateManager.SetStateAsync(BalanceKey, balance);
        // State is batched and committed at end of this turn
    }

    // Read state with default
    public async Task<decimal> GetBalanceAsync()
    {
        return await StateManager.GetOrAddStateAsync(BalanceKey, 0m);
    }

    // Check existence before read
    public async Task<decimal?> TryGetBalanceAsync()
    {
        var result = await StateManager.TryGetStateAsync<decimal>(BalanceKey);
        return result.HasValue ? result.Value : null;
    }

    // Delete state
    public async Task DeleteBalanceAsync()
    {
        await StateManager.RemoveStateAsync(BalanceKey);
    }

    // Save multiple keys in one atomic commit
    public async Task UpdateAccountAsync(decimal newBalance, AccountMetadata metadata)
    {
        // Both writes are committed together at end of the turn
        await StateManager.SetStateAsync(BalanceKey, newBalance);
        await StateManager.SetStateAsync(MetadataKey, metadata);
    }
}
```

## Read and Write Actor State (Python)

```python
from dapr.actor import Actor
from typing import Optional

class AccountActor(Actor):
    BALANCE_KEY = "balance"
    METADATA_KEY = "metadata"

    async def set_balance(self, balance: float):
        await self._state_manager.set_state(self.BALANCE_KEY, balance)

    async def get_balance(self) -> float:
        exists, value = await self._state_manager.try_get_state(self.BALANCE_KEY)
        return value if exists else 0.0

    async def delete_balance(self):
        await self._state_manager.remove_state(self.BALANCE_KEY)

    async def update_account(self, balance: float, metadata: dict):
        await self._state_manager.set_state(self.BALANCE_KEY, balance)
        await self._state_manager.set_state(self.METADATA_KEY, metadata)
        await self._state_manager.save_state()
```

## How State Keys Are Stored

Dapr stores actor state using the following key format in the underlying state store:

```text
{appId}||{actorType}||{actorId}||{stateKey}
```

For example:

```text
my-app||AccountActor||account-001||balance
my-app||AccountActor||account-001||metadata
```

This means you can inspect actor state directly in Redis:

```bash
redis-cli KEYS "my-app||AccountActor||account-001||*"
redis-cli GET "my-app||AccountActor||account-001||balance"
```

## Save State Explicitly (When Needed)

By default, Dapr saves state automatically at the end of each turn. To save state mid-turn (uncommon), call `SaveStateAsync`:

```csharp
public async Task LongRunningOperationAsync()
{
    await StateManager.SetStateAsync("step", "started");
    await StateManager.SaveStateAsync(); // explicit save mid-turn

    await DoExpensiveWork();

    await StateManager.SetStateAsync("step", "completed");
    // auto-saved at end of turn
}
```

## Handle State Migration

When you change your state schema, read both old and new format:

```csharp
public async Task<AccountInfo> GetAccountInfoAsync()
{
    // Try new format first
    var result = await StateManager.TryGetStateAsync<AccountInfo>("account-v2");
    if (result.HasValue) return result.Value;

    // Fall back to old format and migrate
    var oldResult = await StateManager.TryGetStateAsync<OldAccountInfo>("account");
    if (oldResult.HasValue)
    {
        var migrated = new AccountInfo
        {
            Id = oldResult.Value.Id,
            Balance = oldResult.Value.Balance,
            Currency = "USD", // new field with default
            CreatedAt = DateTime.UtcNow
        };
        await StateManager.SetStateAsync("account-v2", migrated);
        await StateManager.RemoveStateAsync("account"); // clean up old key
        return migrated;
    }

    return new AccountInfo(); // brand new actor
}
```

## Best Practices

Keep each state value small and focused:

```csharp
// Prefer: separate keys for separate concerns
await StateManager.SetStateAsync("balance", 100.00m);
await StateManager.SetStateAsync("profile", userProfile);
await StateManager.SetStateAsync("preferences", preferences);

// Avoid: one giant blob that changes frequently
await StateManager.SetStateAsync("everything", new { balance, profile, preferences, history });
```

## Summary

Dapr actor state persistence is handled automatically by the runtime - writes are batched and committed atomically at the end of each actor turn, keyed by actor type and ID in the configured state store. For production, mark exactly one state store component with `actorStateStore: true`, keep state values small and logically separated, and plan for schema migration when evolving your actor state model over time.
