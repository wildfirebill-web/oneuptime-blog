# How to Set Up RabbitMQ Federation Over IPv4 Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RabbitMQ, Federation, IPv4, Messaging, AMQP, Configuration, Distributed System

Description: Learn how to configure RabbitMQ federation to replicate exchanges and queues between RabbitMQ brokers on different IPv4 networks.

---

RabbitMQ federation allows exchanges and queues to be federated - messages published to one broker can flow to another broker on a different IPv4 network without clients needing to know about both brokers. This enables geographic distribution and WAN-friendly message routing.

## How Federation Works

```mermaid
graph LR
    P[Producer\nDatacenter A] --> B1[Broker\n10.0.0.10]
    B1 -->|Federation Link| B2[Broker\n10.1.0.10]
    B2 --> C[Consumer\nDatacenter B]
```

## Enabling the Federation Plugin

```bash
# Enable federation and federation management plugins on both brokers

rabbitmq-plugins enable rabbitmq_federation rabbitmq_federation_management

# Verify the plugins are active
rabbitmq-plugins list | grep federation
```

## Step 1: Define an Upstream

An upstream defines a connection to a remote broker.

```bash
# On the downstream broker (10.1.0.10): define the upstream broker
# The upstream is the broker we want to receive messages FROM
rabbitmqctl set_parameter federation-upstream upstream-dc-a \
  '{"uri":"amqp://feduser:fedpassword@10.0.0.10:5672","max-hops":1,"ack-mode":"on-confirm"}'
```

Parameters:
- `uri` - AMQP connection URI to the upstream broker (IPv4 address)
- `max-hops` - Maximum number of federation hops before a message is dropped
- `ack-mode` - `on-confirm` for reliable delivery

## Step 2: Create a Federation Policy

Policies match exchanges or queues and apply federation behavior.

```bash
# Federate all exchanges matching the pattern "amq.topic" on the default vhost
rabbitmqctl set_policy --apply-to exchanges federate-topic-exchanges "^amq\.topic$" \
  '{"federation-upstream":"upstream-dc-a"}' --priority 10

# Federate a specific exchange named "orders"
rabbitmqctl set_policy --apply-to exchanges federate-orders "^orders$" \
  '{"federation-upstream":"upstream-dc-a"}'
```

## Step 3: Create the Federation User on the Upstream Broker

```bash
# On the upstream broker (10.0.0.10):
rabbitmqctl add_user feduser fedpassword
rabbitmqctl set_permissions feduser "/" ".*" ".*" ".*"
```

## Verifying Federation Links

```bash
# Check the status of all federation links
rabbitmqctl eval 'rabbit_federation_status:status().'

# Or via the management API
curl -u admin:adminpassword http://10.1.0.10:15672/api/federation-links | python3 -m json.tool

# Check via management plugin in the browser
# http://10.1.0.10:15672 → Admin → Federation Status
```

## Testing Federation

```bash
# On the upstream broker: publish a message to the federated exchange
rabbitmqadmin publish exchange=orders routing_key=new-order \
  payload='{"order_id":1}' \
  --host 10.0.0.10 -u admin -p adminpassword

# On the downstream broker: consume the message
rabbitmqadmin get queue=orders-queue \
  --host 10.1.0.10 -u admin -p adminpassword
```

## Key Takeaways

- Federation is one-directional per link; create reverse upstreams for bidirectional flow.
- `max-hops` prevents infinite loops in complex multi-broker topologies.
- The federation user on the upstream broker needs read permissions (`configure`, `write`, `read`).
- Use federation for WAN-distributed systems; use clustering for LAN high-availability setups.
