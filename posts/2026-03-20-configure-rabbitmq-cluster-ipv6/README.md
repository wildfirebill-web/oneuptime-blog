# How to Configure RabbitMQ Cluster with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RabbitMQ, IPv6, Message Broker, Cluster, AMQP, Erlang, DevOps

Description: Configure a RabbitMQ broker cluster to accept AMQP connections and perform cluster communication over IPv6, covering Erlang distribution, listener configuration, and management UI.

---

RabbitMQ is built on Erlang, which has its own networking layer. Configuring RabbitMQ for IPv6 requires both Erlang network settings and RabbitMQ listener configuration.

## RabbitMQ IPv6 Configuration File

```bash
# /etc/rabbitmq/rabbitmq.conf

# Listen on all interfaces including IPv6

listeners.tcp.default = :::5672

# Or specific IPv6 address
# listeners.tcp.1 = 2001:db8::1:5672

# Management plugin on IPv6
management.tcp.port = 15672
management.tcp.ip = ::

# AMQPS (TLS) listener on IPv6
listeners.ssl.default = :::5671

# Cluster configuration
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@[2001:db8::1]
cluster_formation.classic_config.nodes.2 = rabbit@[2001:db8::2]
cluster_formation.classic_config.nodes.3 = rabbit@[2001:db8::3]
```

## Erlang Distribution for IPv6

RabbitMQ's Erlang distribution layer also needs IPv6 configuration:

```bash
# /etc/rabbitmq/advanced.config
[
  {kernel, [
    {inet6, true},          % Enable IPv6 for Erlang distribution
    {inet_default_connect_options, [{nodelay, true}]},
    {inet_default_listen_options, [{nodelay, true}]}
  ]},
  {rabbit, [
    {tcp_listen_options, [
      {backlog, 128},
      {nodelay, true},
      {sndbuf, 32768},
      {recbuf, 32768}
    ]}
  ]}
].
```

## Environment Variables for IPv6

```bash
# /etc/rabbitmq/rabbitmq-env.conf

# Set the ERL_INETRC to use IPv6
export ERL_FLAGS="-kernel inet6 true"

# RabbitMQ node name with IPv6 host
NODENAME=rabbit@$(hostname -f)
```

## Starting RabbitMQ with IPv6

```bash
# Start RabbitMQ
sudo systemctl start rabbitmq-server

# Verify listening on IPv6
ss -tlnp | grep "5672\|15672"
# Should show [::]:5672

# Enable management plugin
sudo rabbitmq-plugins enable rabbitmq_management

# Check RabbitMQ status
sudo rabbitmqctl status

# Check node is accessible over IPv6
sudo rabbitmqctl -n rabbit@2001:db8::1 status
```

## Forming a RabbitMQ Cluster over IPv6

```bash
# On node 2, join the cluster
sudo rabbitmqctl stop_app
sudo rabbitmqctl join_cluster rabbit@[2001:db8::1]
sudo rabbitmqctl start_app

# Verify cluster status
sudo rabbitmqctl cluster_status

# On node 3
sudo rabbitmqctl stop_app
sudo rabbitmqctl join_cluster rabbit@[2001:db8::1]
sudo rabbitmqctl start_app
```

## Connecting to RabbitMQ over IPv6

```python
# Python with pika over IPv6
import pika

# Connect to RabbitMQ via IPv6
credentials = pika.PlainCredentials('guest', 'guest')
parameters = pika.ConnectionParameters(
    host='2001:db8::1',   # IPv6 host (no brackets in pika)
    port=5672,
    credentials=credentials
)

connection = pika.BlockingConnection(parameters)
channel = connection.channel()

# Declare queue
channel.queue_declare(queue='test_ipv6_queue', durable=True)

# Publish message
channel.basic_publish(
    exchange='',
    routing_key='test_ipv6_queue',
    body='Hello from IPv6 producer!'
)

print("Message sent over IPv6")
connection.close()
```

## TLS for RabbitMQ over IPv6

```bash
# /etc/rabbitmq/rabbitmq.conf
listeners.ssl.default = :::5671

ssl_options.cacertfile = /etc/ssl/certs/ca.crt
ssl_options.certfile   = /etc/ssl/certs/rabbitmq.crt
ssl_options.keyfile    = /etc/ssl/private/rabbitmq.key
ssl_options.verify     = verify_peer
ssl_options.fail_if_no_peer_cert = false
```

## Firewall Rules for RabbitMQ IPv6

```bash
# AMQP
sudo ip6tables -A INPUT -p tcp --dport 5672 -j ACCEPT
# AMQPS (TLS)
sudo ip6tables -A INPUT -p tcp --dport 5671 -j ACCEPT
# Management UI
sudo ip6tables -A INPUT -p tcp --dport 15672 -j ACCEPT
# Erlang distribution
sudo ip6tables -A INPUT -p tcp --dport 25672 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 4369 -j ACCEPT   # EPMD

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

RabbitMQ's combination of Erlang distribution settings and AMQP listener configuration provides complete IPv6 support for message broker clusters, enabling event-driven architectures on IPv6 networks.
