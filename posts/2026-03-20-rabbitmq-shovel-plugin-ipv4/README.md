# How to Configure RabbitMQ Shovel Plugin for IPv4 Remote Brokers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RabbitMQ, Shovel, IPv4, AMQP, Messaging, Plugin, Configuration

Description: Learn how to configure the RabbitMQ Shovel plugin to move messages from a queue on one IPv4 broker to an exchange on another broker.

---

The RabbitMQ Shovel plugin consumes messages from a source queue and republishes them to a destination exchange on the same or a different broker. Unlike federation (which is topology-based), a shovel moves specific messages from point A to point B, making it ideal for message migration, aggregation, or routing between datacenters.

## Use Cases for Shovel

- Move messages from a local broker to a central aggregator over IPv4.
- Migrate messages from an old broker to a new one without downtime.
- Forward overflow messages from a local queue to a backup broker.

## Enabling the Shovel Plugin

```bash
rabbitmq-plugins enable rabbitmq_shovel rabbitmq_shovel_management
systemctl restart rabbitmq-server
```

## Static Shovel via rabbitmq.conf

```ini
# /etc/rabbitmq/rabbitmq.conf

# Define a static shovel (configured at startup)

shovel.my_shovel.source.protocol = amqp091
shovel.my_shovel.source.uris.1   = amqp://admin:password@127.0.0.1:5672
shovel.my_shovel.source.queue    = source-queue

shovel.my_shovel.dest.protocol = amqp091
shovel.my_shovel.dest.uris.1   = amqp://shoveler:shovelerpass@10.0.0.20:5672
shovel.my_shovel.dest.exchange  = target-exchange
shovel.my_shovel.dest.exchange_key = shovel.forwarded

# Delete source messages after successful delivery
shovel.my_shovel.ack_mode = on-confirm
```

## Dynamic Shovel via CLI

Dynamic shovels are stored in the RabbitMQ database and can be created/deleted at runtime.

```bash
# Create a dynamic shovel on the local broker
rabbitmqctl set_parameter shovel my-dynamic-shovel \
'{
  "src-protocol": "amqp091",
  "src-uri": "amqp://admin:password@127.0.0.1:5672",
  "src-queue": "local-queue",
  "dest-protocol": "amqp091",
  "dest-uri": "amqp://shoveler:shovelerpass@10.0.0.20:5672",
  "dest-exchange": "remote-exchange",
  "dest-exchange-key": "forwarded",
  "ack-mode": "on-confirm",
  "reconnect-delay": 5
}'
```

## Creating the Destination User on the Remote Broker

```bash
# On the remote broker (10.0.0.20):
rabbitmqctl add_user shoveler shovelerpass
rabbitmqctl set_permissions shoveler "/" ".*" ".*" ".*"
```

## Checking Shovel Status

```bash
# Via CLI
rabbitmqctl shovel_status

# Via management API
curl -u admin:password http://localhost:15672/api/shovels | python3 -m json.tool

# Expected state: "running" when the shovel is active
```

## Removing a Dynamic Shovel

```bash
rabbitmqctl clear_parameter shovel my-dynamic-shovel
```

## Shovel vs Federation

| Feature | Shovel | Federation |
|---------|--------|------------|
| Level | Queue-based | Exchange or queue |
| Direction | Source → Dest (one-way) | Bidirectional (with policies) |
| Messages | Moves (consumes from source) | Copies (source retains) |
| Configuration | Per-queue | Topology-level policies |
| Best for | Migration, aggregation | Geo-distribution |

## Key Takeaways

- Shovels move messages (consuming from source) while federation copies them.
- Dynamic shovels (via `set_parameter`) can be created and removed without restart.
- Set `reconnect-delay` to auto-reconnect if the destination IPv4 broker becomes temporarily unavailable.
- Use `ack-mode: on-confirm` for reliable delivery; the message is only deleted from the source after the destination confirms receipt.
