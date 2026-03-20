# How to Configure RabbitMQ Listeners on a Specific IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RabbitMQ, IPv4, Listeners, Configuration, AMQP, Messaging

Description: Configure RabbitMQ to listen on specific IPv4 addresses for AMQP and management traffic using rabbitmq.conf, restricting exposure to intended network interfaces.

## Introduction

RabbitMQ defaults to listening on all interfaces (0.0.0.0) for AMQP (port 5672) and management (port 15672). On multi-homed servers, binding to a specific IPv4 address limits exposure to the intended network interface.

## Configuration

```bash
# /etc/rabbitmq/rabbitmq.conf

# Bind AMQP listener to specific IPv4

listeners.tcp.1 = 10.0.0.5:5672

# Also bind to localhost for local connections
listeners.tcp.2 = 127.0.0.1:5672

# Management plugin binding (see management section)
management.listener.ip = 10.0.0.5
management.listener.port = 15672
```

## Advanced Binding with rabbitmq.conf

```bash
# /etc/rabbitmq/rabbitmq.conf

# Multiple listeners on different interfaces/ports
listeners.tcp.1 = 127.0.0.1:5672
listeners.tcp.2 = 10.0.0.5:5672
listeners.tcp.3 = 10.0.0.5:5673   # Second AMQP port (optional)

# Disable all default bindings (use only specified)
# Note: specifying listeners.tcp overrides the default 0.0.0.0:5672

# Network settings
tcp_listen_options.backlog = 128
tcp_listen_options.nodelay = true
```

## Erlang-Style Configuration (advanced.config)

```erlang
%% /etc/rabbitmq/advanced.config
%% Use when rabbitmq.conf doesn't support the needed options

[
  {rabbit, [
    {tcp_listeners, [{"10.0.0.5", 5672}, {"127.0.0.1", 5672}]},
    {ssl_listeners, [{"10.0.0.5", 5671}]}
  ]}
].
```

## Verifying the Configuration

```bash
# Test rabbitmq.conf syntax
sudo rabbitmqctl eval 'rabbit:start().'

# Restart RabbitMQ
sudo systemctl restart rabbitmq-server

# Check what ports RabbitMQ is listening on
sudo ss -tlnp | grep -E "beam|rabbitmq|:5672|:15672"
# Expected: 10.0.0.5:5672 and 127.0.0.1:5672

# List active listeners via RabbitMQ management
sudo rabbitmq-diagnostics listeners
# Shows: {address, port, protocol, description}
```

## Testing Connections

```bash
# Test AMQP connection on specific IP
python3 -c "
import pika
params = pika.ConnectionParameters('10.0.0.5', 5672, '/',
    pika.PlainCredentials('guest', 'guest'))
conn = pika.BlockingConnection(params)
print('Connected to', conn.params.host)
conn.close()
"

# Test with rabbitmq CLI
rabbitmqctl -n rabbit@10.0.0.5 status

# Test that old IP is NOT accessible
nc -zv 0.0.0.0 5672   # Should fail
nc -zv 10.0.0.5 5672  # Should succeed
```

## Conclusion

Specify RabbitMQ listeners with `listeners.tcp.N = ip:port` in `rabbitmq.conf`. This overrides the default 0.0.0.0:5672 binding. Always include `127.0.0.1:5672` to preserve local connections for management tools. Verify with `rabbitmq-diagnostics listeners` and `ss -tlnp` after restarting. For management plugin binding, use `management.listener.ip`.
