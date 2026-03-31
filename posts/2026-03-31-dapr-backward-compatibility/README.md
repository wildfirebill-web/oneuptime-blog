# How to Implement Backward Compatibility in Dapr Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Backward Compatibility, Versioning, API, Design

Description: Learn how to implement backward compatibility in Dapr services using additive changes, field deprecation patterns, and versioned endpoints to avoid breaking consumers.

---

## Overview

Backward compatibility in Dapr microservices means existing consumers continue to work without changes after a service is updated. Breaking changes - removing fields, changing response types, or modifying message schemas - can cause cascading failures in a distributed system. This guide covers practical patterns for maintaining backward compatibility.

## Principles of Backward Compatible Changes

Changes that are safe (backward compatible):
- Adding new optional request fields
- Adding new response fields
- Adding new endpoints
- Widening accepted value ranges

Changes that are breaking:
- Removing request or response fields
- Renaming fields
- Changing field types
- Removing endpoint routes
- Changing pub/sub message schemas

## Additive Changes in Request Handling

Always treat unexpected request fields as ignorable and provide defaults for new fields:

```javascript
app.post('/orders', async (req, res) => {
  const {
    customerId,
    items,
    // New optional field - backward compatible
    priority = 'standard',
    // New optional field with default
    notificationEmail = null
  } = req.body;

  const order = await createOrder({
    customerId,
    items,
    priority,
    notificationEmail
  });

  res.status(201).json(order);
});
```

## Versioned Response Objects

Return all fields, including deprecated ones, to avoid breaking existing consumers:

```javascript
function formatOrderResponse(order, apiVersion = 'v1') {
  const base = {
    orderId: order.id,
    status: order.status,
    createdAt: order.createdAt,
    items: order.items
  };

  if (apiVersion === 'v2') {
    return {
      ...base,
      // New v2 fields
      trackingId: order.trackingId,
      estimatedDelivery: order.estimatedDelivery
    };
  }

  // v1 response - never remove fields that v1 consumers depend on
  return {
    ...base,
    // Keep deprecated field for v1 compatibility
    orderDate: order.createdAt  // deprecated in favor of createdAt
  };
}
```

## Deprecating Pub/Sub Message Fields

When evolving pub/sub message schemas, keep old fields while adding new ones:

```javascript
// Old message shape still supported
const legacyMessage = {
  orderId: '123',
  userId: 'u-456'  // deprecated
};

// New message shape
const newMessage = {
  orderId: '123',
  userId: 'u-456',       // kept for backward compat
  customerId: 'cust-456', // new canonical field
  version: 2
};

// Consumer handles both versions
app.post('/orders/created', (req, res) => {
  const event = req.body.data;
  // Support both old and new field names
  const customerId = event.customerId || event.userId;
  processOrder(event.orderId, customerId);
  res.sendStatus(200);
});
```

## Using Feature Flags for Gradual Rollout

Introduce new behavior behind a feature flag to test backward compatibility:

```javascript
const { DaprClient } = require('@dapr/dapr');

async function getFeatureFlag(flagName) {
  const client = new DaprClient();
  const state = await client.state.get('redis-statestore', `feature:${flagName}`);
  return state === 'true';
}

app.post('/orders', async (req, res) => {
  const useNewProcessor = await getFeatureFlag('new-order-processor');

  const order = useNewProcessor
    ? await newOrderProcessor(req.body)
    : await legacyOrderProcessor(req.body);

  res.status(201).json(order);
});
```

## Deprecation Headers in HTTP Responses

Signal deprecated endpoints to consumers via standard headers:

```javascript
app.get('/v1/orders', (req, res) => {
  res.set('Deprecation', 'true');
  res.set('Sunset', 'Sat, 31 Dec 2026 23:59:59 GMT');
  res.set('Link', '</v2/orders>; rel="successor-version"');

  // Still process the v1 request normally
  const orders = getOrdersV1();
  res.json(orders);
});
```

## Testing Backward Compatibility in CI

Run backward compatibility tests against the previous version of the service contract:

```bash
# Download previous contract version
curl -o ./tests/contracts/v1-contract.yaml \
  https://pact-broker.example.com/contracts/order-service/v1

# Validate current implementation against old contract
npx @stoplight/prism-cli mock ./contracts/openapi.yaml &
npx dredd ./tests/contracts/v1-contract.yaml http://localhost:4010
```

## Summary

Backward compatibility in Dapr services requires treating API changes as additive only, providing defaults for new fields, keeping deprecated fields in responses, and handling both old and new pub/sub message shapes in subscribers. Feature flags enable gradual rollout of breaking changes, and deprecation headers communicate timelines to consumers. Running backward compatibility tests in CI catches regressions before deployment.
