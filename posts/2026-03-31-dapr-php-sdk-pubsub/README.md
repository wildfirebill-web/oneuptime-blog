# How to Use Dapr PHP SDK for Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, PHP, Pub/Sub, Messaging, Event-Driven

Description: Learn how to publish and subscribe to events in PHP applications using the Dapr PHP SDK, with attribute-based topic registration and event handling.

---

## Introduction

Dapr Pub/Sub enables PHP microservices to communicate asynchronously through a message broker. The Dapr PHP SDK provides attributes and a routing mechanism to register topic subscriptions and publish events using the `DaprClient`. This guide covers both publishing and subscribing patterns.

## Prerequisites

```bash
composer require dapr/php-sdk
dapr init
```

## Publishing an Event

Use `DaprClient::tryPublishEvent` to publish to any configured pubsub component:

```php
<?php
require_once 'vendor/autoload.php';

use Dapr\Client\DaprClient;

$client = DaprClient::clientBuilder()->build();

$orderPayload = [
    'order_id' => 'ORD-001',
    'item'     => 'widget',
    'quantity' => 5,
    'amount'   => 49.95
];

$client->publishEvent(
    pubsubName: 'pubsub',
    topicName: 'orders',
    data: $orderPayload,
    metadata: ['content-type' => 'application/json']
);

echo "Published order ORD-001\n";
```

## Subscribing with Attribute-Based Routing

The PHP SDK supports attribute-based subscription using the `#[Topic]` attribute on controller methods:

```php
<?php
use Dapr\Attributes\FromBody;
use Dapr\PubSub\Subscribe;
use Dapr\PubSub\Topic;

class OrderController {
    #[Subscribe]
    #[Topic(pubsub: 'pubsub', name: 'orders')]
    public function handleOrder(#[FromBody] array $order): \Dapr\PubSub\CloudEvent {
        echo "Received order: " . $order['order_id'] . "\n";
        $this->processOrder($order);
        return \Dapr\PubSub\CloudEvent::success();
    }

    private function processOrder(array $order): void {
        // Process the order
        echo "Processing item: " . $order['item'] . " x" . $order['quantity'] . "\n";
    }
}
```

## Manual Subscription Registration

For frameworks without attribute support, register subscriptions via the `/dapr/subscribe` endpoint:

```php
<?php
// router.php
$path = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);

if ($path === '/dapr/subscribe' && $_SERVER['REQUEST_METHOD'] === 'GET') {
    header('Content-Type: application/json');
    echo json_encode([
        [
            'pubsubname' => 'pubsub',
            'topic'      => 'orders',
            'route'      => '/handle-order'
        ]
    ]);
    exit;
}

if ($path === '/handle-order' && $_SERVER['REQUEST_METHOD'] === 'POST') {
    $body = json_decode(file_get_contents('php://input'), true);
    $order = $body['data'] ?? [];
    echo "Processing: " . json_encode($order) . "\n";
    header('Content-Type: application/json');
    echo json_encode(['status' => 'SUCCESS']);
    exit;
}
```

## Dead Letter Topics

Configure a dead letter topic for failed messages:

```php
<?php
// Registration with dead letter
$subscriptions = [
    [
        'pubsubname'      => 'pubsub',
        'topic'           => 'orders',
        'route'           => '/handle-order',
        'deadLetterTopic' => 'orders-dlq'
    ]
];
```

## Running the PHP App

```bash
dapr run \
  --app-id php-pubsub \
  --app-port 8080 \
  --components-path ./components \
  -- php -S 0.0.0.0:8080 router.php
```

## Testing Pub/Sub

```bash
dapr publish \
  --publish-app-id php-pubsub \
  --pubsub pubsub \
  --topic orders \
  --data '{"order_id":"ORD-001","item":"widget","quantity":5}'
```

## Summary

The Dapr PHP SDK supports both attribute-based and manual subscription registration. Publishing events requires only a `DaprClient` instance and a call to `publishEvent`. The CloudEvent success/retry/drop responses control message acknowledgment behavior, giving you fine-grained control over message processing semantics.
