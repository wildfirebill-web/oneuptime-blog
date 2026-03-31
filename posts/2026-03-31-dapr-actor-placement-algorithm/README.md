# How to Understand Dapr Actor Placement Algorithm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Placement, Algorithm, Architecture

Description: Understand how Dapr's consistent hash ring placement algorithm distributes virtual actors across application instances and how this affects actor activation and rebalancing.

---

Dapr's virtual actor model guarantees that only one instance of a given actor ID runs at any time, regardless of how many application replicas exist. This guarantee is enforced by the placement algorithm, which uses a consistent hash ring to deterministically map actor IDs to application instances.

## The Consistent Hash Ring

The placement service builds a consistent hash ring from all registered application instances. Each instance occupies multiple virtual nodes on the ring to improve distribution uniformity.

When an actor is invoked:
1. The calling sidecar asks the placement service where actor OrderActor/order-123 is hosted
2. The placement service hashes the key to find its position on the ring
3. The sidecar that owns the nearest ring position hosts the actor
4. The placement service returns the target sidecar's address
5. The calling sidecar forwards the invocation to the target

## Actor ID Hashing

The hash input is the combination of actor type and actor ID. This means:
- The same actor type with different IDs may be placed on different instances
- Related actors can be co-located by using a common prefix in their IDs
- Actor type distribution across the ring is determined by the virtual node positions of each instance

```go
// Simplified hash placement logic
hash := fnv1a(actorType + "/" + actorId)
position := hash % ringSize
instance := ring.FindOwner(position)
```

## Virtual Nodes for Uniform Distribution

Each application instance claims multiple positions on the ring (virtual nodes). The default virtual node count balances distribution across instances.

With 3 application replicas and 100 virtual nodes per instance, the ring has 300 total positions. Each instance owns roughly one third of the ring.

## What Happens When an Instance Joins

When a new application instance starts:
1. Its Dapr sidecar registers with the placement service
2. The placement service assigns virtual node positions to the new instance
3. A new actor table is generated and disseminated to all sidecars
4. Actors previously owned by other instances that now hash to the new instance are deactivated
5. On next invocation, those actors are re-activated on the new host

## What Happens When an Instance Leaves

When an application instance shuts down:
1. Its sidecar deregisters from the placement service
2. The actor table is updated to remove the leaving instance
3. Actors owned by the leaving instance are redistributed to remaining instances
4. The new owners re-activate actors on their next invocation

## Actor Co-location

To keep related actors on the same instance, use a consistent ID prefix:

```go
// These IDs tend to hash to the same ring segment
orderActorID := "cust-456-order-789"
paymentActorID := "cust-456-payment-012"
```

This is probabilistic, not guaranteed.

## Observing Placement in Action

Watch actor activation and deactivation in sidecar logs:

```bash
kubectl logs my-app -c daprd | grep -E "actor.*activat|actor.*deactivat|placement"
```

Check which actors are currently active on an instance:

```bash
curl http://localhost:3500/v1.0/metadata | jq '.activeActorsCount'
```

## Handling Rebalancing

Configure retries to handle transient errors during rebalancing after a rolling deployment:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: actor-resiliency
spec:
  policies:
    retries:
      actorDefault:
        policy: constant
        duration: 250ms
        maxRetries: 3
  targets:
    actors:
      OrderActor:
        retry: actorDefault
```

## Summary

Dapr's consistent hash ring placement algorithm deterministically maps actor IDs to application instances, ensuring single-instance activation while enabling transparent failover and rebalancing. Understanding how virtual nodes and re-registration work helps you design actor IDs for good distribution and configure appropriate retry policies for rebalancing events.
