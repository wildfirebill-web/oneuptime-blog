# How to Use Dapr PHP SDK for State Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, PHP, State Management, Redis, Microservice

Description: Learn how to save, retrieve, update, and delete state in PHP applications using the Dapr PHP SDK with transactional and bulk state operations.

---

## Introduction

Dapr State Management provides PHP applications with a consistent API to store and retrieve key-value data across different backends such as Redis, PostgreSQL, and Azure Cosmos DB. The Dapr PHP SDK makes these operations straightforward with typed state classes and transactions.

## Prerequisites

```bash
composer require dapr/php-sdk
dapr init
```

## Basic State Operations

```php
<?php
require_once 'vendor/autoload.php';

use Dapr\Client\DaprClient;

$client = DaprClient::clientBuilder()->build();

// Save state
$client->trySaveState(
    storeName: 'statestore',
    key: 'product-001',
    value: ['name' => 'Widget', 'price' => 9.99, 'stock' => 100]
);

// Get state
$state = $client->tryGetState(
    storeName: 'statestore',
    key: 'product-001',
    asType: 'array'
);
echo "Product: " . $state->value['name'] . "\n";

// Delete state
$client->tryDeleteState(
    storeName: 'statestore',
    key: 'product-001'
);
echo "Deleted product-001\n";
```

## Using State Objects with Typed Classes

Define typed state classes for cleaner code:

```php
<?php
use Dapr\State\Attributes\StateStore;

#[StateStore('statestore', \Dapr\Consistency\EventualLastWrite::class)]
class ProductState extends \Dapr\State\AppState {
    public string $name = '';
    public float $price = 0.0;
    public int $stock = 0;
}
```

## Bulk State Operations

Retrieve multiple keys in one call:

```php
<?php
$keys = ['product-001', 'product-002', 'product-003'];
$states = $client->tryGetBulkState(
    storeName: 'statestore',
    keys: $keys,
    asType: 'array'
);

foreach ($states as $state) {
    if ($state->value !== null) {
        echo $state->key . ": " . $state->value['name'] . "\n";
    }
}
```

## Transactional State Updates

Use transactions to atomically update multiple keys:

```php
<?php
use Dapr\Client\DaprClient;
use Dapr\State\TransactionalState;

$client = DaprClient::clientBuilder()->build();

$operations = [
    [
        'operation' => 'upsert',
        'request' => [
            'key' => 'order-001',
            'value' => json_encode(['status' => 'shipped'])
        ]
    ],
    [
        'operation' => 'upsert',
        'request' => [
            'key' => 'inventory-widget',
            'value' => json_encode(['stock' => 97])
        ]
    ]
];

$client->executeStateTransaction('statestore', $operations);
echo "Transaction committed\n";
```

## Optimistic Concurrency with ETags

```php
<?php
$state = $client->tryGetState(
    storeName: 'statestore',
    key: 'counter',
    asType: 'int'
);

$currentEtag = $state->etag;
$newValue = ($state->value ?? 0) + 1;

// Only save if the value has not changed since we read it
$client->trySaveState(
    storeName: 'statestore',
    key: 'counter',
    value: $newValue,
    etag: $currentEtag,
    concurrency: 'first-write'
);
```

## Running the App

```bash
dapr run \
  --app-id php-state-app \
  --components-path ./components \
  -- php state_demo.php
```

## Summary

The Dapr PHP SDK provides save, get, delete, bulk, and transactional state operations through `DaprClient`. Typed state classes keep your code organized, while ETag-based optimistic concurrency prevents lost updates under concurrent writes. Swapping the backend from Redis to PostgreSQL only requires updating the component YAML.
