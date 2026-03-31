# How to Build Dapr Actors with PHP SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actors, PHP, Virtual Actors, Distributed Systems

Description: Learn how to implement Dapr virtual actors using the PHP SDK, including defining actor interfaces, managing state, and invoking actors from PHP applications.

---

## Overview of Dapr Actors in PHP

Dapr's PHP SDK supports the virtual actor pattern, allowing PHP applications to host stateful actors that are automatically managed by the Dapr runtime. Actors get activated on first call, their state persists automatically, and the runtime handles placement across nodes.

## Installing the Dapr PHP SDK

```bash
composer require dapr/php-sdk
```

## Project Structure

```text
/actor-app
  /src
    CartActor.php
    CartActorInterface.php
  /public
    index.php
  composer.json
  components/
    statestore.yaml
```

## Defining an Actor Interface

```php
<?php

namespace App\Actors;

use Dapr\Actors\IActor;

interface CartActorInterface extends IActor
{
    public function addItem(string $productId, int $quantity): void;
    public function removeItem(string $productId): void;
    public function getItems(): array;
    public function getTotal(): float;
    public function checkout(): array;
}
```

## Implementing the Actor

```php
<?php

namespace App\Actors;

use Dapr\Actors\Actor;
use Dapr\Actors\ActorState;

class CartActor extends Actor implements CartActorInterface
{
    protected static string $dapr_type = 'CartActor';

    private array $items = [];

    public function __construct(string $id, ActorState $state)
    {
        parent::__construct($id, $state);
        $this->items = $state->get('items') ?? [];
    }

    public function addItem(string $productId, int $quantity): void
    {
        if (isset($this->items[$productId])) {
            $this->items[$productId]['quantity'] += $quantity;
        } else {
            $this->items[$productId] = [
                'product_id' => $productId,
                'quantity'   => $quantity,
                'price'      => $this->getProductPrice($productId),
            ];
        }
        $this->state->set('items', $this->items);
        echo "Added {$quantity} of {$productId} to cart {$this->id}\n";
    }

    public function removeItem(string $productId): void
    {
        unset($this->items[$productId]);
        $this->state->set('items', $this->items);
    }

    public function getItems(): array
    {
        return array_values($this->items);
    }

    public function getTotal(): float
    {
        return array_sum(array_map(
            fn($item) => $item['price'] * $item['quantity'],
            $this->items
        ));
    }

    public function checkout(): array
    {
        $total = $this->getTotal();
        $items = $this->items;

        // Clear the cart after checkout
        $this->items = [];
        $this->state->set('items', []);

        return [
            'order_id' => uniqid('ORD-'),
            'items'    => array_values($items),
            'total'    => $total,
            'status'   => 'confirmed',
        ];
    }

    private function getProductPrice(string $productId): float
    {
        // In a real app, fetch from a catalog service
        $prices = ['WIDGET-1' => 9.99, 'GADGET-2' => 24.99, 'THING-3' => 4.99];
        return $prices[$productId] ?? 0.0;
    }
}
```

## Setting Up the Actor Host

PHP actors run as a web application. Set up the entry point to handle Dapr actor requests:

```php
<?php
// public/index.php

require_once __DIR__ . '/../vendor/autoload.php';

use Dapr\App;
use App\Actors\CartActor;

$app = App::create(
    configure: fn(\DI\ContainerBuilder $builder) => $builder->addDefinitions([
        'dapr.actors' => [CartActor::class],
    ])
);

$app->run();
```

## Invoking Actors from Client Code

Use the Dapr HTTP API directly or the PHP SDK client to invoke actors:

```php
<?php

use Dapr\Actors\ActorProxy;
use App\Actors\CartActorInterface;

// Create a proxy for a specific cart (actor ID = user ID)
$cart = ActorProxy::create(CartActorInterface::class, 'user-1001');

// Add items to the cart
$cart->addItem('WIDGET-1', 2);
$cart->addItem('GADGET-2', 1);

// Get cart contents
$items = $cart->getItems();
echo "Cart items: " . json_encode($items) . "\n";

// Get total
$total = $cart->getTotal();
echo "Cart total: $" . number_format($total, 2) . "\n";

// Checkout
$order = $cart->checkout();
echo "Order placed: " . json_encode($order) . "\n";
```

## Using the HTTP API to Invoke Actors

If you prefer the HTTP API over the SDK proxy:

```bash
# Add item to cart actor for user-1001
curl -X PUT http://localhost:3500/v1.0/actors/CartActor/user-1001/method/addItem \
  -H "Content-Type: application/json" \
  -d '{"productId": "WIDGET-1", "quantity": 2}'

# Get cart items
curl http://localhost:3500/v1.0/actors/CartActor/user-1001/method/getItems

# Checkout
curl -X PUT http://localhost:3500/v1.0/actors/CartActor/user-1001/method/checkout
```

## State Component Configuration

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
    value: localhost:6379
  - name: actorStateStore
    value: "true"
```

## Running the PHP Actor Application

```bash
# Install dependencies
composer install

# Start the actor service with Dapr
dapr run --app-id cart-service \
         --app-port 8080 \
         --dapr-http-port 3500 \
         --resources-path ./components \
         -- php -S 0.0.0.0:8080 -t public
```

## Summary

The Dapr PHP SDK enables PHP applications to leverage the virtual actor model for managing distributed state. Define actor behavior in PHP classes, use the built-in `ActorState` for automatic persistence, and expose actors via the standard Dapr HTTP interface. This makes it practical to adopt Dapr's actor pattern in existing PHP applications without requiring a full rewrite in another language.
