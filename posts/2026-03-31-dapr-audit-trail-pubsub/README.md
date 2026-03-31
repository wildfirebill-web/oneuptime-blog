# How to Implement Audit Trail with Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Audit Trail, Compliance, Security

Description: Learn how to implement a centralized audit trail using Dapr pub/sub to capture and store all significant actions across your microservices for compliance and forensics.

---

## Why Use Dapr Pub/Sub for Audit Trails?

An audit trail must be tamper-evident, complete, and low-latency. Dapr pub/sub decouples audit logging from your business services - services publish audit events and a dedicated audit consumer persists them. This ensures auditing never blocks the critical path.

## Configure the Pub/Sub Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: audit-pubsub
  namespace: production
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: "kafka:9092"
    - name: consumerGroup
      value: "audit-consumers"
```

## Audit Event Envelope

Standardize the envelope for all audit events:

```javascript
function createAuditEvent(opts) {
  return {
    eventId: require('crypto').randomUUID(),
    eventType: 'audit',
    action: opts.action,           // 'user.login', 'order.deleted', etc.
    actorId: opts.actorId,         // user or service that performed the action
    actorType: opts.actorType,     // 'user' | 'service' | 'system'
    resourceType: opts.resourceType,
    resourceId: opts.resourceId,
    outcome: opts.outcome,         // 'success' | 'failure'
    metadata: opts.metadata || {},
    occurredAt: new Date().toISOString(),
    sourceService: process.env.DAPR_APP_ID
  };
}
```

## Publishing Audit Events from Services

```javascript
const { DaprClient } = require('@dapr/dapr');
const client = new DaprClient();

async function deleteOrder(orderId, requestingUserId) {
  // Business logic
  await db.query('DELETE FROM orders WHERE id = $1', [orderId]);

  // Publish audit event asynchronously - does not block response
  await client.pubsub.publish('audit-pubsub', 'audit.event', createAuditEvent({
    action: 'order.deleted',
    actorId: requestingUserId,
    actorType: 'user',
    resourceType: 'order',
    resourceId: orderId,
    outcome: 'success',
    metadata: { reason: 'user-requested' }
  }));
}
```

## Audit Consumer - Persist to Immutable Store

```javascript
const { DaprServer } = require('@dapr/dapr');
const server = new DaprServer();

await server.pubsub.subscribe('audit-pubsub', 'audit.event', async (event) => {
  // Write to append-only table - no updates or deletes allowed
  await db.query(
    `INSERT INTO audit_log
     (event_id, action, actor_id, actor_type, resource_type, resource_id,
      outcome, metadata, occurred_at, source_service)
     VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)`,
    [
      event.eventId, event.action, event.actorId, event.actorType,
      event.resourceType, event.resourceId, event.outcome,
      JSON.stringify(event.metadata), event.occurredAt, event.sourceService
    ]
  );
});
```

## Querying the Audit Log

```sql
-- All actions on a specific order
SELECT * FROM audit_log
WHERE resource_type = 'order' AND resource_id = '12345'
ORDER BY occurred_at DESC;

-- All actions by a specific user in the last 24 hours
SELECT * FROM audit_log
WHERE actor_id = 'user-789'
  AND occurred_at >= NOW() - INTERVAL '24 hours'
ORDER BY occurred_at DESC;

-- Failed login attempts
SELECT * FROM audit_log
WHERE action = 'user.login' AND outcome = 'failure'
ORDER BY occurred_at DESC LIMIT 100;
```

## Ensuring Completeness with Dead Letter Topics

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: audit-subscription
spec:
  topic: audit.event
  routes:
    default: /audit/ingest
  pubsubname: audit-pubsub
  deadLetterTopic: audit-dead-letter
```

Alert on any messages landing in the dead letter topic to ensure no audit events are lost.

## Summary

Dapr pub/sub provides a reliable, non-blocking audit trail mechanism. Business services fire-and-forget audit events onto the pub/sub bus, while a dedicated consumer persists them to an append-only store. Dead letter topics catch any failed deliveries, ensuring auditability for compliance requirements.
