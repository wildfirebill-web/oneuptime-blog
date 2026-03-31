# How to Build Dapr Actors with PHP SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, PHP, Actors, Microservices, Distributed Systems

Description: Learn how to implement Dapr virtual actors in PHP using the Dapr PHP SDK, covering actor registration, state management, and HTTP API integration.

---

Dapr Actors provide a turn-based concurrency model that simplifies building distributed stateful services. While PHP is traditionally used for stateless web applications, the Dapr PHP SDK makes it possible to use the actor model in PHP applications. This guide demonstrates how to define, register, and operate Dapr actors using PHP.

## Setting Up the PHP Project

Install the Dapr PHP SDK using Composer.

```bash
composer require dapr/php-sdk
```

Create a project structure:

```text
dapr-actors-php/
  composer.json
  public/
    index.php
  src/
    Actors/
      CounterActor.php
      CounterActorInterface.php
    bootstrap.php
  dapr/
    components/
      statestore.yaml
```

Configure Composer autoloading:

```json
{
  "require": {
    "dapr/php-sdk": "^1.1"
  },
  "autoload": {
    "psr-4": {
      "App\\": "src/"
    }
  }
}
```

## Defining the Actor Interface

The actor interface declares the methods that callers can invoke on the actor.

```php
<?php

namespace App\Actors;

use Dapr\Actors\Attributes\DaprType;

#[DaprType('CounterActor')]
interface CounterActorInterface
{
    /**
     * Increment the counter by the given amount.
     */
    public function increment(int $amount): void;

    /**
     * Get the current counter value.
     */
    public function getCount(): int;

    /**
     * Reset the counter to zero.
     */
    public function reset(): void;

    /**
     * Get the counter history.
     */
    public function getHistory(): array;
}
```

## Implementing the Actor Class

Implement the interface by extending `Actor` and using the actor's state manager for persistence.

```php
<?php

namespace App\Actors;

use Dapr\Actors\Actor;
use Dapr\Actors\Attributes\DaprType;
use Dapr\Actors\ActorState;

#[DaprType('CounterActor')]
class CounterActor extends Actor implements CounterActorInterface
{
    // Actor state is automatically persisted by Dapr
    private int $count = 0;
    private array $history = [];

    public function onActivate(): void
    {
        // Called when the actor is first activated
        // Load state from Dapr state store
        $savedCount = $this->stateManager->get('count', 0);
        $savedHistory = $this->stateManager->get('history', []);

        $this->count = $savedCount;
        $this->history = $savedHistory;

        echo "Actor {$this->id} activated with count {$this->count}" . PHP_EOL;
    }

    public function increment(int $amount): void
    {
        if ($amount <= 0) {
            throw new \InvalidArgumentException('Amount must be positive');
        }

        $this->count += $amount;
        $this->history[] = [
            'action' => 'increment',
            'amount' => $amount,
            'timestamp' => date('Y-m-d H:i:s'),
            'newCount' => $this->count
        ];

        // Persist updated state
        $this->stateManager->set('count', $this->count);
        $this->stateManager->set('history', $this->history);

        echo "Counter incremented by {$amount}, new count: {$this->count}" . PHP_EOL;
    }

    public function getCount(): int
    {
        return $this->count;
    }

    public function reset(): void
    {
        $this->history[] = [
            'action' => 'reset',
            'previousCount' => $this->count,
            'timestamp' => date('Y-m-d H:i:s'),
            'newCount' => 0
        ];
        $this->count = 0;

        $this->stateManager->set('count', $this->count);
        $this->stateManager->set('history', $this->history);
    }

    public function getHistory(): array
    {
        return $this->history;
    }

    public function onDeactivate(): void
    {
        // Called when the actor is about to be deactivated
        echo "Actor {$this->id} deactivating" . PHP_EOL;
    }
}
```

## Registering Actors and Starting the HTTP Server

Dapr invokes PHP actors via HTTP. You need to create an HTTP endpoint that the Dapr runtime can call.

```php
<?php
// public/index.php

require __DIR__ . '/../vendor/autoload.php';

use Dapr\App;
use App\Actors\CounterActor;

$app = App::create(
    configuration: [],
    register: function(\DI\ContainerBuilder $builder) {
        // Register actor types
        $builder->addDefinitions([]);
    }
);

// Register the actor with Dapr
$app->register_actor(CounterActor::class);

// Add custom routes for direct API access
$app->post('/counter/{actorId}/increment', function(
    string $actorId,
    \Psr\Http\Message\RequestInterface $request
) use ($app) {
    $body = json_decode((string) $request->getBody(), true);
    $amount = $body['amount'] ?? 1;

    // Use the Dapr actor proxy to call the actor
    $proxy = $app->get_actor_proxy(CounterActorInterface::class, $actorId);
    $proxy->increment($amount);

    return ['success' => true, 'actorId' => $actorId];
});

$app->get('/counter/{actorId}', function(
    string $actorId
) use ($app) {
    $proxy = $app->get_actor_proxy(CounterActorInterface::class, $actorId);
    $count = $proxy->getCount();
    $history = $proxy->getHistory();

    return [
        'actorId' => $actorId,
        'count' => $count,
        'history' => $history
    ];
});

$app->start();
```

Run the PHP application with Dapr:

```bash
dapr run \
  --app-id counter-actor \
  --app-port 8080 \
  --dapr-http-port 3500 \
  -- php -S 0.0.0.0:8080 public/index.php
```

## Calling Actors via HTTP API

You can call actor methods directly through the Dapr HTTP API without using the PHP SDK proxy.

```bash
# Increment a counter actor
curl -X POST http://localhost:3500/v1.0/actors/CounterActor/user-123/method/increment \
  -H "Content-Type: application/json" \
  -d '{"amount": 5}'

# Get the current count
curl http://localhost:3500/v1.0/actors/CounterActor/user-123/method/getCount

# Get history
curl http://localhost:3500/v1.0/actors/CounterActor/user-123/method/getHistory

# Reset the counter
curl -X POST http://localhost:3500/v1.0/actors/CounterActor/user-123/method/reset
```

Interact with actor state directly (useful for debugging):

```bash
# Read actor state
curl http://localhost:3500/v1.0/actors/CounterActor/user-123/state/count

# Write actor state (not recommended in production - use actor methods instead)
curl -X PUT http://localhost:3500/v1.0/actors/CounterActor/user-123/state \
  -H "Content-Type: application/json" \
  -d '[{"operation":"upsert","request":{"key":"count","value":0}}]'
```

## Working with Timers and Reminders

PHP actors support timers and reminders by registering callbacks that the Dapr runtime invokes.

```php
<?php

namespace App\Actors;

use Dapr\Actors\Actor;
use Dapr\Actors\Attributes\DaprType;

#[DaprType('SessionActor')]
class SessionActor extends Actor
{
    public function onActivate(): void
    {
        // Register a reminder that fires after 30 minutes
        $this->createReminder('session-expiry', [
            'dueTime' => '0h30m0s',
            'period' => '',  // fire once
            'data' => base64_encode(json_encode(['reason' => 'inactivity']))
        ]);

        // Register a timer for heartbeats (not persisted)
        $this->createTimer('heartbeat', [
            'dueTime' => '0h1m0s',
            'period' => '0h5m0s',
            'callback' => 'heartbeatCallback',
            'data' => null
        ]);
    }

    public function heartbeatCallback(): void
    {
        $lastActivity = $this->stateManager->get('last_activity', time());
        echo "Heartbeat for actor {$this->id}, last active: " . date('H:i:s', $lastActivity) . PHP_EOL;
        $this->stateManager->set('last_activity', time());
    }

    public function receiveReminder(string $reminderName, mixed $data): void
    {
        if ($reminderName === 'session-expiry') {
            echo "Session {$this->id} expired" . PHP_EOL;
            $this->stateManager->set('status', 'expired');
            // Delete the timer as the session is done
            $this->deleteTimer('heartbeat');
        }
    }

    public function ping(): void
    {
        // Update last activity time to prevent expiry
        $this->stateManager->set('last_activity', time());
    }
}
```

## Summary

Building Dapr Actors with the PHP SDK brings the power of the virtual actor pattern to the PHP ecosystem. You learned how to define actor interfaces and implement actor classes with persistent state management, register actors in the Dapr runtime via an HTTP server, call actor methods using both the PHP SDK proxy and the Dapr HTTP API, and add scheduled behavior with timers and reminders. While PHP is often used for stateless web applications, Dapr Actors open the door to building stateful, concurrent distributed services with familiar PHP code.
