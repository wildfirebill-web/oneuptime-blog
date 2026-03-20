# How to Configure Apache Kafka with IPv6 Listeners

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Kafka, Apache Kafka, Message Queues, Streaming, Event Streaming

Description: Learn how to configure Apache Kafka brokers to listen on IPv6 addresses, advertise IPv6 endpoints to clients, and form clusters with IPv6 inter-broker communication.

## Kafka broker.properties IPv6 Configuration

```properties
# /etc/kafka/server.properties (or /opt/kafka/config/server.properties)

# Broker ID

broker.id=1

# Listeners - use [::] for all IPv6 interfaces
listeners=PLAINTEXT://[2001:db8::10]:9092

# Advertised listeners (what clients use to connect)
advertised.listeners=PLAINTEXT://[2001:db8::10]:9092

# Inter-broker listener
inter.broker.listener.name=PLAINTEXT

# ZooKeeper connection with IPv6
zookeeper.connect=[2001:db8::10]:2181,[2001:db8::11]:2181,[2001:db8::12]:2181/kafka

# Log directories
log.dirs=/var/lib/kafka/logs
```

## Listen on All IPv6 Interfaces

```properties
# Listen on all interfaces (both IPv4 and IPv6)
listeners=PLAINTEXT://[::]:9092

# Or use separate listeners for each protocol
listeners=PLAINTEXT://[2001:db8::10]:9092,PLAINTEXT_IPV4://0.0.0.0:9093
listener.security.protocol.map=PLAINTEXT:PLAINTEXT,PLAINTEXT_IPV4:PLAINTEXT
```

## KRaft Mode (Kafka without ZooKeeper)

```properties
# /etc/kafka/server.properties (KRaft mode)

# Node roles
process.roles=broker,controller

# Node ID
node.id=1

# Controller quorum voters - IPv6 addresses
controller.quorum.voters=1@[2001:db8::10]:9093,2@[2001:db8::11]:9093,3@[2001:db8::12]:9093

# Listeners
listeners=PLAINTEXT://[2001:db8::10]:9092,CONTROLLER://[2001:db8::10]:9093
advertised.listeners=PLAINTEXT://[2001:db8::10]:9092

# Listener name to security protocol map
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
inter.broker.listener.name=PLAINTEXT
controller.listener.names=CONTROLLER

# Log directory
log.dirs=/var/lib/kafka/kraft-combined-logs
```

## Start and Verify

```bash
# Start Kafka (traditional)
systemctl start kafka

# Check listening ports
ss -6 -tlnp | grep java
# Expected: [2001:db8::10]:9092

# List topics using IPv6 broker
kafka-topics.sh --bootstrap-server [2001:db8::10]:9092 --list

# Create a test topic
kafka-topics.sh --bootstrap-server [2001:db8::10]:9092 \
    --create --topic test-ipv6 --partitions 3 --replication-factor 1
```

## Produce and Consume over IPv6

```bash
# Produce messages (console producer)
kafka-console-producer.sh --bootstrap-server [2001:db8::10]:9092 --topic test-ipv6

# Consume messages (console consumer)
kafka-console-consumer.sh --bootstrap-server [2001:db8::10]:9092 \
    --topic test-ipv6 --from-beginning

# Describe cluster
kafka-broker-api-versions.sh --bootstrap-server [2001:db8::10]:9092
```

## Python Kafka Client over IPv6

```python
from kafka import KafkaProducer, KafkaConsumer

# Producer connecting via IPv6
producer = KafkaProducer(
    bootstrap_servers=['[2001:db8::10]:9092', '[2001:db8::11]:9092'],
    value_serializer=lambda v: v.encode('utf-8')
)

# Send a message
producer.send('test-ipv6', value='Hello from IPv6!')
producer.flush()
producer.close()

# Consumer connecting via IPv6
consumer = KafkaConsumer(
    'test-ipv6',
    bootstrap_servers=['[2001:db8::10]:9092'],
    auto_offset_reset='earliest',
    group_id='ipv6-consumer-group',
    value_deserializer=lambda m: m.decode('utf-8')
)

for message in consumer:
    print(f"Received: {message.value} from partition {message.partition}")
```

## JVM IPv6 Configuration

```bash
# Kafka runs on JVM - configure for IPv6
# /etc/kafka/kafka-env.sh or set in systemd service

export KAFKA_OPTS="-Djava.net.preferIPv6Addresses=true $KAFKA_OPTS"

# Or add to KAFKA_JVM_PERFORMANCE_OPTS in kafka-server-start.sh
```

## Summary

Configure Kafka for IPv6 by setting `listeners=PLAINTEXT://[2001:db8::10]:9092` and `advertised.listeners=PLAINTEXT://[2001:db8::10]:9092` in `server.properties`. Use `[::]:9092` to listen on all interfaces. For KRaft mode, specify IPv6 addresses in `controller.quorum.voters`. Connect clients using `[2001:db8::10]:9092` in bootstrap-servers (with bracket notation). Add `-Djava.net.preferIPv6Addresses=true` to JVM options. Verify with `ss -6 -tlnp | grep java` and `kafka-topics.sh --bootstrap-server [2001:db8::10]:9092 --list`.
