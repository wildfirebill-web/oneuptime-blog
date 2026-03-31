# How to Implement Actor Pipelines in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Pipeline, Workflow, Processing

Description: Learn how to implement actor pipelines in Dapr where each stage actor processes data and passes it to the next stage, enabling composable, observable processing chains.

---

## What Is an Actor Pipeline?

An actor pipeline is a processing chain where each stage is an actor. Data flows from stage to stage: stage 1 enriches the payload, stage 2 validates it, stage 3 transforms it, stage 4 persists it. Each stage is independently scalable, testable, and observable.

## Pipeline Design

```text
raw-event -> [EnrichmentActor] -> [ValidationActor] -> [TransformActor] -> [StorageActor]
```

Each stage actor:
1. Receives a payload from the previous stage.
2. Processes it.
3. Invokes the next stage actor.

## Stage 1: Enrichment Actor

```javascript
class EnrichmentActor {
  constructor(host) {
    this.stateManager = host.stateManager;
    this.client = new (require('@dapr/dapr').DaprClient)();
  }

  async process(payload) {
    // Enrich with user details
    const user = await this.client.invokeMethod(
      'user-service', 'users/' + payload.userId, 'GET'
    );

    const enriched = {
      ...payload,
      userEmail: user.email,
      userPlan: user.plan,
      enrichedAt: Date.now()
    };

    // Pass to validation stage
    const nextActorId = `validate:${payload.eventId}`;
    await this.client.actor.invoke('ValidationActor', nextActorId, 'process', enriched);
    return { success: true };
  }
}
```

## Stage 2: Validation Actor

```javascript
class ValidationActor {
  constructor(host) {
    this.client = new (require('@dapr/dapr').DaprClient)();
  }

  async process(payload) {
    const errors = [];

    if (!payload.userId) errors.push('Missing userId');
    if (!payload.eventType) errors.push('Missing eventType');
    if (!payload.userEmail) errors.push('Enrichment failed - no email');

    if (errors.length > 0) {
      await this.client.pubsub.publish('events-pubsub', 'pipeline.validation-failed', {
        eventId: payload.eventId,
        errors
      });
      return { success: false, errors };
    }

    // Pass to transform stage
    const nextActorId = `transform:${payload.eventId}`;
    await this.client.actor.invoke('TransformActor', nextActorId, 'process', payload);
    return { success: true };
  }
}
```

## Stage 3: Transform Actor

```javascript
class TransformActor {
  constructor(host) {
    this.client = new (require('@dapr/dapr').DaprClient)();
  }

  async process(payload) {
    const transformed = {
      event_id: payload.eventId,
      event_type: payload.eventType,
      user_id: payload.userId,
      user_email: payload.userEmail,
      user_plan: payload.userPlan,
      occurred_at: new Date(payload.timestamp).toISOString(),
      processed_at: new Date().toISOString()
    };

    // Pass to storage stage
    const nextActorId = `store:${payload.eventId}`;
    await this.client.actor.invoke('StorageActor', nextActorId, 'process', transformed);
    return { success: true };
  }
}
```

## Stage 4: Storage Actor

```javascript
class StorageActor {
  async process(payload) {
    await db.query(
      `INSERT INTO events (event_id, event_type, user_id, user_email, user_plan, occurred_at, processed_at)
       VALUES ($1, $2, $3, $4, $5, $6, $7)`,
      [payload.event_id, payload.event_type, payload.user_id,
       payload.user_email, payload.user_plan, payload.occurred_at, payload.processed_at]
    );
    return { success: true, stored: true };
  }
}
```

## Starting the Pipeline

```javascript
const { DaprClient } = require('@dapr/dapr');
const client = new DaprClient();

async function submitToPipeline(event) {
  const actorId = `enrich:${event.eventId}`;
  await client.actor.invoke('EnrichmentActor', actorId, 'process', event);
}
```

## Summary

Actor pipelines in Dapr create composable, sequential processing chains where each stage is independently deployable and testable. Because each pipeline run uses unique actor IDs per event, stages process in parallel across different events while maintaining strict sequential ordering within a single event's pipeline execution.
