# How to Use Custom Serializers in Dapr PHP SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, PHP, Serialization, Custom Serializer, SDK

Description: Learn how to implement and register custom serializers in the Dapr PHP SDK to control how your data objects are serialized and deserialized for state and pub/sub.

---

## Introduction

By default, the Dapr PHP SDK uses JSON serialization for state and pub/sub payloads. Custom serializers let you control exactly how your domain objects are encoded and decoded, which is useful for legacy data formats, performance optimization, or integration with systems that expect specific schemas.

## Prerequisites

```bash
composer require dapr/php-sdk
```

## Default Serialization Behavior

The SDK serializes objects to JSON automatically:

```php
<?php
use Dapr\Client\DaprClient;

$client = DaprClient::clientBuilder()->build();

// This is serialized as {"name":"Widget","price":9.99}
$client->trySaveState('statestore', 'product-001', [
    'name'  => 'Widget',
    'price' => 9.99
]);
```

## Implementing a Custom Serializer

Create a class that implements `\Dapr\Serialization\ISerialize`:

```php
<?php
// src/Serializers/ProductSerializer.php
namespace App\Serializers;

use Dapr\Serialization\ISerialize;

class ProductSerializer implements ISerialize {
    public function serialize(mixed $data): string {
        if (is_array($data)) {
            // Custom format: pipe-delimited
            return implode('|', [
                $data['sku']   ?? '',
                $data['name']  ?? '',
                (string)($data['price'] ?? 0),
                (string)($data['stock'] ?? 0)
            ]);
        }
        return json_encode($data);
    }

    public function deserialize(string $data, string $type): mixed {
        $parts = explode('|', $data);
        return [
            'sku'   => $parts[0] ?? '',
            'name'  => $parts[1] ?? '',
            'price' => (float)($parts[2] ?? 0),
            'stock' => (int)($parts[3] ?? 0)
        ];
    }
}
```

## Registering the Custom Serializer

```php
<?php
use Dapr\Client\DaprClient;
use App\Serializers\ProductSerializer;

$client = DaprClient::clientBuilder()
    ->withSerializer(new ProductSerializer())
    ->build();

$client->trySaveState('statestore', 'product-SKU-001', [
    'sku'   => 'SKU-001',
    'name'  => 'Widget Pro',
    'price' => 19.99,
    'stock' => 250
]);
```

## Custom Deserializer for State Reads

```php
<?php
$state = $client->tryGetState(
    storeName: 'statestore',
    key: 'product-SKU-001',
    asType: 'array'
);

$product = $state->value;
echo "Product: {$product['name']} - \${$product['price']}\n";
echo "Stock: {$product['stock']}\n";
```

## Using PHP-DI to Register Serializers

With PHP-DI, register the serializer in the container:

```php
<?php
use DI\ContainerBuilder;
use Dapr\Serialization\ISerialize;
use App\Serializers\ProductSerializer;

$builder = new ContainerBuilder();
$builder->addDefinitions([
    ISerialize::class => \DI\create(ProductSerializer::class)
]);
$container = $builder->build();
```

## Custom CloudEvent Serializer for Pub/Sub

```php
<?php
namespace App\Serializers;

class OrderEventSerializer implements \Dapr\Serialization\ISerialize {
    public function serialize(mixed $data): string {
        return json_encode([
            'v'  => 1,
            'id' => $data['order_id'],
            'ts' => time(),
            'pl' => $data
        ]);
    }

    public function deserialize(string $data, string $type): mixed {
        $envelope = json_decode($data, true);
        return $envelope['pl'] ?? [];
    }
}
```

## Summary

Custom serializers in the Dapr PHP SDK let you control the wire format of your state and pub/sub payloads. Implement `ISerialize` with `serialize` and `deserialize` methods, then register it via the client builder or PHP-DI. This is especially useful when integrating with existing systems that require specific data formats or when optimizing for payload size.
