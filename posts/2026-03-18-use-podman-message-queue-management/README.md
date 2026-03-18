# How to Use Podman for Message Queue Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Message Queues, RabbitMQ, Kafka, Redis, Containers

Description: Learn how to deploy and manage message queue systems like RabbitMQ, Apache Kafka, and Redis with Podman containers for reliable asynchronous communication between services.

---

> Podman simplifies message queue deployment by running brokers like RabbitMQ and Kafka in isolated containers with persistent storage, making it easy to set up reliable messaging infrastructure.

Message queues are the backbone of distributed systems, handling asynchronous communication between services. Deploying and managing message brokers traditionally requires careful installation and configuration. Podman containers package the entire broker environment, making it reproducible and portable across development and production environments.

This guide covers deploying RabbitMQ, Apache Kafka, and Redis as message brokers using Podman, including clustering, monitoring, and production configuration.

---

## Why Podman for Message Queues

Message brokers need to be reliable and isolated from other services. Podman's rootless containers limit the impact of any broker vulnerability. Pods let you group a broker with its management UI and monitoring tools. Named volumes ensure message persistence across restarts, and Quadlet integration makes brokers behave like native system services.

## Deploying RabbitMQ

RabbitMQ is one of the most popular message brokers. Deploy it with the management plugin enabled:

```bash
podman volume create rabbitmq-data

podman run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=securepass \
  -v rabbitmq-data:/var/lib/rabbitmq:Z \
  docker.io/library/rabbitmq:3-management-alpine
```

Access the management UI at `http://localhost:15672`. The AMQP port is `5672` for client connections.

## RabbitMQ Custom Configuration

Create a custom configuration for production use:

```bash
cat > ~/rabbitmq/rabbitmq.conf << 'EOF'
listeners.tcp.default = 5672
management.listener.port = 15672

vm_memory_high_watermark.relative = 0.7
disk_free_limit.absolute = 1GB

log.console = true
log.console.level = info

consumer_timeout = 1800000
EOF

cat > ~/rabbitmq/definitions.json << 'EOF'
{
  "queues": [
    {"name": "orders", "vhost": "/", "durable": true, "arguments": {}},
    {"name": "notifications", "vhost": "/", "durable": true, "arguments": {}},
    {"name": "dead-letters", "vhost": "/", "durable": true, "arguments": {}}
  ],
  "exchanges": [
    {"name": "app-events", "vhost": "/", "type": "topic", "durable": true}
  ],
  "bindings": [
    {"source": "app-events", "vhost": "/", "destination": "orders", "destination_type": "queue", "routing_key": "order.*"}
  ]
}
EOF

podman run -d \
  --name rabbitmq-prod \
  -p 5672:5672 -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=securepass \
  -v ~/rabbitmq/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro,Z \
  -v ~/rabbitmq/definitions.json:/etc/rabbitmq/definitions.json:ro,Z \
  -v rabbitmq-data:/var/lib/rabbitmq:Z \
  docker.io/library/rabbitmq:3-management-alpine
```

## Deploying Apache Kafka

Kafka requires ZooKeeper (or KRaft for newer versions). Deploy both in a pod:

```bash
podman pod create --name kafka-pod \
  -p 9092:9092 \
  -p 2181:2181

# ZooKeeper
podman run -d --pod kafka-pod \
  --name zookeeper \
  -e ZOOKEEPER_CLIENT_PORT=2181 \
  -v zk-data:/var/lib/zookeeper/data:Z \
  docker.io/confluentinc/cp-zookeeper:7.6.0

# Kafka
podman run -d --pod kafka-pod \
  --name kafka \
  -e KAFKA_BROKER_ID=1 \
  -e KAFKA_ZOOKEEPER_CONNECT=127.0.0.1:2181 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  -e KAFKA_LOG_RETENTION_HOURS=168 \
  -v kafka-data:/var/lib/kafka/data:Z \
  docker.io/confluentinc/cp-kafka:7.6.0
```

## Kafka with KRaft Mode (No ZooKeeper)

Newer Kafka versions support KRaft mode, eliminating the ZooKeeper dependency:

```bash
podman run -d \
  --name kafka-kraft \
  -p 9092:9092 \
  -e KAFKA_NODE_ID=1 \
  -e KAFKA_PROCESS_ROLES=broker,controller \
  -e KAFKA_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS=1@localhost:9093 \
  -e CLUSTER_ID=$(podman run --rm docker.io/confluentinc/cp-kafka:7.6.0 kafka-storage random-uuid) \
  -v kafka-kraft-data:/var/lib/kafka/data:Z \
  docker.io/confluentinc/cp-kafka:7.6.0
```

## Creating Kafka Topics

```bash
podman exec kafka kafka-topics \
  --create \
  --topic orders \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

podman exec kafka kafka-topics \
  --list \
  --bootstrap-server localhost:9092

podman exec kafka kafka-topics \
  --describe \
  --topic orders \
  --bootstrap-server localhost:9092
```

## Redis as a Message Broker

Redis supports pub/sub and Streams for message queuing:

```bash
podman run -d \
  --name redis-queue \
  -p 6379:6379 \
  -v redis-data:/data:Z \
  docker.io/library/redis:7-alpine \
  redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
```

Test Redis Streams:

```bash
# Add messages to a stream
podman exec redis-queue redis-cli XADD orders '*' product "widget" quantity "5"
podman exec redis-queue redis-cli XADD orders '*' product "gadget" quantity "3"

# Read messages
podman exec redis-queue redis-cli XRANGE orders - +

# Create consumer group
podman exec redis-queue redis-cli XGROUP CREATE orders processors $ MKSTREAM
```

## Producer and Consumer Examples

A Python producer for RabbitMQ:

```python
# producer.py
import pika
import json

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost', 5672,
        credentials=pika.PlainCredentials('admin', 'securepass'))
)
channel = connection.channel()
channel.queue_declare(queue='orders', durable=True)

message = {'order_id': 123, 'product': 'widget', 'quantity': 5}
channel.basic_publish(
    exchange='',
    routing_key='orders',
    body=json.dumps(message),
    properties=pika.BasicProperties(delivery_mode=2)  # persistent
)
print(f"Sent: {message}")
connection.close()
```

A Python consumer:

```python
# consumer.py
import pika
import json

def process_order(ch, method, properties, body):
    order = json.loads(body)
    print(f"Processing order: {order}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost', 5672,
        credentials=pika.PlainCredentials('admin', 'securepass'))
)
channel = connection.channel()
channel.queue_declare(queue='orders', durable=True)
channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='orders', on_message_callback=process_order)

print("Waiting for messages...")
channel.start_consuming()
```

## Monitoring Message Queues

Add a Kafka UI for monitoring:

```bash
podman run -d \
  --name kafka-ui \
  --network host \
  -e KAFKA_CLUSTERS_0_NAME=local \
  -e KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=localhost:9092 \
  docker.io/provectuslabs/kafka-ui:latest
```

Check RabbitMQ queue status from the command line:

```bash
podman exec rabbitmq rabbitmqctl list_queues name messages consumers
podman exec rabbitmq rabbitmqctl status
```

## Running as systemd Services

```ini
# ~/.config/containers/systemd/rabbitmq.container
[Container]
Image=docker.io/library/rabbitmq:3-management-alpine
PublishPort=5672:5672
PublishPort=15672:15672
Environment=RABBITMQ_DEFAULT_USER=admin
Environment=RABBITMQ_DEFAULT_PASS=securepass
Volume=rabbitmq-data:/var/lib/rabbitmq:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```

## Conclusion

Podman makes message queue deployment and management straightforward through containerization. Whether you choose RabbitMQ for traditional message queuing, Kafka for high-throughput event streaming, or Redis for lightweight pub/sub, Podman provides consistent isolation, persistent storage, and systemd integration. The rootless execution model adds security to broker deployments that often handle sensitive application data.
