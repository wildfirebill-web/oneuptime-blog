# How to Scale Dapr Actors to Millions of Instances

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Scalability, Kubernetes, Placement

Description: Scale Dapr Actor deployments to millions of instances by tuning the placement service, state store partitioning, and actor idle timeout settings.

---

## Dapr Actor Scalability Model

Each Dapr Actor instance is a lightweight virtual object. Millions of instances can exist because only active actors consume memory - idle actors are deactivated and their state is persisted in the state store. The placement service distributes actor instances across pods using consistent hashing.

## State Store for High Scale

Choose a state store that supports millions of keys efficiently:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: actor-statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis-cluster.default.svc.cluster.local:6379"
  - name: actorStateStore
    value: "true"
  - name: keyPrefix
    value: "actors"
  - name: maxRetries
    value: "3"
```

For very large deployments use Redis Cluster or Azure Cosmos DB:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: actor-statestore
spec:
  type: state.azure.cosmosdb
  version: v1
  metadata:
  - name: url
    value: "https://mycosmosdb.documents.azure.com:443/"
  - name: masterKey
    secretKeyRef:
      name: cosmos-secret
      key: key
  - name: database
    value: "dapr-actors"
  - name: collection
    value: "actor-state"
  - name: actorStateStore
    value: "true"
```

## Tuning Actor Idle Timeout

Aggressive idle timeouts reduce memory usage when scaling to millions:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddActors(options =>
    {
        options.Actors.RegisterActor<DeviceActor>();

        // Deactivate idle actors after 5 minutes
        options.ActorIdleTimeout = TimeSpan.FromMinutes(5);

        // Scan for idle actors every 30 seconds
        options.ActorScanInterval = TimeSpan.FromSeconds(30);

        // Drain actors before pod shutdown
        options.DrainOngoingCallTimeout = TimeSpan.FromSeconds(60);
        options.DrainRebalancedActors = true;

        // Reentrancy for recursive actor calls
        options.ReentrancyConfig = new ActorReentrancyConfig { Enabled = true };
    });
}
```

## Scaling the Placement Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dapr-placement-server
  namespace: dapr-system
spec:
  replicas: 3  # Run 3 placement servers for HA
```

```bash
# Helm install with HA placement
helm upgrade dapr dapr/dapr \
  --set dapr_placement.replicaCount=3 \
  --set dapr_placement.ha.enabled=true \
  -n dapr-system
```

## Batch Actor Activation

For preloading millions of actors at startup:

```python
import asyncio
import aiohttp

DAPR_PORT = 3500
ACTOR_TYPE = "DeviceActor"

async def activate_actor(session: aiohttp.ClientSession, actor_id: str):
    url = f"http://localhost:{DAPR_PORT}/v1.0/actors/{ACTOR_TYPE}/{actor_id}/state"
    try:
        async with session.get(url) as resp:
            return actor_id, resp.status
    except Exception as e:
        return actor_id, str(e)

async def activate_batch(actor_ids: list[str], concurrency: int = 500):
    connector = aiohttp.TCPConnector(limit=concurrency)
    async with aiohttp.ClientSession(connector=connector) as session:
        tasks = [activate_actor(session, aid) for aid in actor_ids]
        results = await asyncio.gather(*tasks)
    return results

# Activate 1 million actors in batches
async def main():
    batch_size = 10000
    for i in range(0, 1_000_000, batch_size):
        ids = [f"device-{i+j}" for j in range(batch_size)]
        results = await activate_batch(ids)
        print(f"Activated batch {i//batch_size}: {len(results)} actors")

asyncio.run(main())
```

## Monitoring Actor Scale

```bash
# Check placement service connected hosts
kubectl logs -n dapr-system deploy/dapr-placement-server | grep -i "host added"

# Check actor count via Dapr metrics
kubectl port-forward -n dapr-system svc/dapr-placement-server 9090:9090
curl http://localhost:9090/metrics | grep dapr_placement_actor_count
```

## Summary

Scaling Dapr Actors to millions of instances requires three things: a state store that handles millions of keys (Redis Cluster or Cosmos DB), aggressive idle timeout tuning to deactivate unused actors, and a highly available placement service. The virtual actor model means you never manage actor lifecycle manually - Dapr activates and deactivates them automatically based on demand.
