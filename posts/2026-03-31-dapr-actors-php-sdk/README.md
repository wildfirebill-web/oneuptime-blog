# How to Use Dapr Actors with PHP SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, PHP, Actor, Distributed Computing, State Management

Description: Learn how to implement Dapr Virtual Actors in PHP using the PHP SDK, including actor interfaces, state persistence, timers, and reminders.

---

## Introduction

Dapr Virtual Actors implement the actor model for distributed systems. Each actor is a single-threaded, stateful object identified by a unique ID. The Dapr PHP SDK provides interfaces and traits to implement actors that can be invoked from any Dapr-enabled service. This guide builds a simple counter actor with state, timers, and reminders.

## Prerequisites

```bash
composer require dapr/php-sdk
dapr init
```

## Defining the Actor Interface

```php
<?php
// src/Actors/CounterInterface.php
namespace App\Actors;

interface CounterInterface extends \Dapr\Actors\IActor {
    public function increment(): int;
    public function decrement(): int;
    public function getCount(): int;
    public function reset(): void;
}
```

## Implementing the Actor

```php
<?php
// src/Actors/Counter.php
namespace App\Actors;

use Dapr\Actors\Actor;
use Dapr\Actors\Attributes\DaprType;

#[DaprType('Counter')]
class Counter extends Actor implements CounterInterface {
    private int $count = 0;

    public function onActivate(): void {
        // Load state on activation
        $saved = $this->stateManager->tryGet('count', 0);
        $this->count = $saved;
        echo "Counter {$this->id->id} activated with count={$this->count}\n";
    }

    public function increment(): int {
        $this->count++;
        $this->stateManager->set('count', $this->count);
        return $this->count;
    }

    public function decrement(): int {
        $this->count = max(0, $this->count - 1);
        $this->stateManager->set('count', $this->count);
        return $this->count;
    }

    public function getCount(): int {
        return $this->count;
    }

    public function reset(): void {
        $this->count = 0;
        $this->stateManager->set('count', 0);
    }
}
```

## Registering the Actor

```php
<?php
// src/app.php
use Dapr\App;

$app = App::create();
$app->register_actor(Counter::class);
$app->start();
```

## Adding Timers and Reminders

```php
<?php
// Inside the Counter actor class

public function startTimer(): void {
    $this->registerTimer(
        name: 'log-timer',
        callback: 'logCount',
        due_time: new \DateInterval('PT5S'),
        period: new \DateInterval('PT30S')
    );
}

public function logCount(): void {
    echo "Timer fired - current count: {$this->count}\n";
}

public function startReminder(): void {
    $this->registerReminder(
        name: 'daily-reset',
        data: null,
        due_time: new \DateInterval('P1D'),
        period: new \DateInterval('P1D')
    );
}

public function receiveReminder(string $reminderName, mixed $data): void {
    if ($reminderName === 'daily-reset') {
        $this->reset();
        echo "Counter reset by daily reminder\n";
    }
}
```

## Invoking the Actor from Another Service

```php
<?php
use Dapr\Actors\ActorProxy;
use App\Actors\CounterInterface;

$proxy = ActorProxy::create(CounterInterface::class, 'counter-1');
$value = $proxy->increment();
echo "New count: {$value}\n";

$current = $proxy->getCount();
echo "Current count: {$current}\n";
```

## Running the Actor App

```bash
dapr run \
  --app-id counter-actors \
  --app-port 8080 \
  -- php -S 0.0.0.0:8080 src/app.php
```

## Summary

Dapr Virtual Actors in PHP provide a simple programming model for stateful distributed objects. Each actor handles one request at a time, eliminating race conditions. State is automatically persisted to the configured state store, and reminders survive actor deactivation. The PHP SDK's actor proxy lets any service invoke actor methods transparently across the cluster.
