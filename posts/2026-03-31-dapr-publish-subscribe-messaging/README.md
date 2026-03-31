# How to Implement Publish-Subscribe Messaging with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Messaging, Event-Driven, Microservice

Description: A practical guide to implementing publish-subscribe messaging with Dapr, covering component setup, publishing events, and subscribing with both programmatic and declarative approaches.

---

Publish-subscribe messaging is a core pattern in event-driven microservices. Dapr abstracts the message broker, letting you switch between Redis, Kafka, RabbitMQ, and cloud brokers without code changes. This guide walks through a complete Pub/Sub implementation.

## Install and Configure Dapr

```bash
# Install Dapr CLI
wget -q https://raw.githubusercontent.com/dapr/cli/master/install/install.sh -O - | /bin/bash

# Initialize Dapr locally
dapr init

# Verify installation
dapr --version
```

## Define the Pub/Sub Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: messagebus
  namespace: default
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: "localhost:6379"
  - name: redisPassword
    value: ""
```

## Publisher Service

```javascript
const express = require('express');
const axios = require('axios');

const app = express();
app.use(express.json());

const DAPR_HTTP_PORT = process.env.DAPR_HTTP_PORT || 3500;

app.post('/publish', async (req, res) => {
    const { topic, message } = req.body;

    try {
        await axios.post(
            `http://localhost:${DAPR_HTTP_PORT}/v1.0/publish/messagebus/${topic}`,
            message
        );
        res.json({ success: true, topic, message });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

app.listen(3000, () => console.log('Publisher running on port 3000'));
```

## Subscriber Service - Programmatic Subscription

```javascript
const express = require('express');
const app = express();
app.use(express.json());

// Dapr calls this endpoint to discover subscriptions
app.get('/dapr/subscribe', (req, res) => {
    res.json([
        {
            pubsubname: 'messagebus',
            topic: 'orders',
            route: '/orders'
        },
        {
            pubsubname: 'messagebus',
            topic: 'notifications',
            route: '/notifications'
        }
    ]);
});

app.post('/orders', (req, res) => {
    const order = req.body;
    console.log('Received order:', JSON.stringify(order, null, 2));
    res.json({ status: 'SUCCESS' });
});

app.post('/notifications', (req, res) => {
    const notification = req.body;
    console.log('Received notification:', notification);
    res.json({ status: 'SUCCESS' });
});

app.listen(3001, () => console.log('Subscriber running on port 3001'));
```

## Declarative Subscription (YAML)

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: orders-sub
  namespace: default
spec:
  pubsubname: messagebus
  topic: orders
  routes:
    default: /orders
  scopes:
  - order-processor
```

## Running the Application

```bash
# Start the subscriber
dapr run --app-id order-processor \
  --app-port 3001 \
  --components-path ./components \
  -- node subscriber.js

# Start the publisher
dapr run --app-id order-publisher \
  --app-port 3000 \
  --components-path ./components \
  -- node publisher.js

# Publish a test message
curl -X POST http://localhost:3000/publish \
  -H "Content-Type: application/json" \
  -d '{"topic":"orders","message":{"id":"ORD-001","total":99.99}}'
```

## Summary

Dapr Pub/Sub provides a broker-agnostic messaging layer that supports both programmatic and declarative subscriptions. Publishers use a simple HTTP POST to Dapr's sidecar, while subscribers expose endpoints that Dapr discovers automatically. Switching message brokers requires only a component YAML change, not application code changes.
