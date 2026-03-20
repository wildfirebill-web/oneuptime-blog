# How to Deploy Apache Pulsar via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Apache Pulsar, Messaging, Docker, Streaming

Description: Deploy Apache Pulsar distributed messaging and streaming platform using Portainer for event-driven architectures.

## Introduction

Apache Pulsar is a cloud-native distributed messaging and streaming platform that supports multi-tenancy, geo-replication, and persistent message storage. This guide deploys a standalone Pulsar instance via Portainer Stacks for development and small production workloads.

## Prerequisites

- Portainer installed with Docker
- At least 4 GB RAM (Pulsar standalone includes broker, bookkeeper, and ZooKeeper)
- Basic understanding of publish-subscribe messaging

## Step 1: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack** and use the following configuration:

```yaml
# docker-compose.yml - Apache Pulsar Standalone
version: "3.8"

services:
  pulsar:
    image: apachepulsar/pulsar:3.2.0
    container_name: pulsar
    restart: unless-stopped
    command: bin/pulsar standalone
    ports:
      - "6650:6650"    # Pulsar binary protocol
      - "8080:8080"    # HTTP admin API
    volumes:
      - pulsar_data:/pulsar/data
      - pulsar_conf:/pulsar/conf
    environment:
      - PULSAR_MEM=-Xms512m -Xmx512m -XX:MaxDirectMemorySize=256m
    healthcheck:
      test: ["CMD", "bin/pulsar-admin", "brokers", "healthcheck"]
      interval: 30s
      timeout: 15s
      retries: 5
      start_period: 60s
    networks:
      - pulsar_net

volumes:
  pulsar_data:
  pulsar_conf:

networks:
  pulsar_net:
    driver: bridge
```

## Step 2: Verify Deployment

```bash
# Check the broker is ready
curl http://localhost:8080/admin/v2/brokers/ready

# List tenants
curl http://localhost:8080/admin/v2/tenants

# Check topics
curl http://localhost:8080/admin/v2/persistent/public/default
```

## Step 3: Produce and Consume Messages

Test using the built-in CLI tools:

```bash
# Produce a message
docker exec pulsar bin/pulsar-client produce \
  persistent://public/default/test-topic \
  --messages "Hello from Pulsar!"

# Consume messages
docker exec pulsar bin/pulsar-client consume \
  persistent://public/default/test-topic \
  --subscription-name test-sub \
  --num-messages 10
```

## Step 4: Python Client Example

```python
# pip install pulsar-client
import pulsar

client = pulsar.Client('pulsar://localhost:6650')

# Create a producer
producer = client.create_producer('persistent://public/default/my-topic')
producer.send(b'Hello, Pulsar!')
producer.close()

# Create a consumer
consumer = client.subscribe(
    'persistent://public/default/my-topic',
    subscription_name='my-subscription'
)
msg = consumer.receive(timeout_millis=5000)
print(f"Received: {msg.data()}")
consumer.acknowledge(msg)
consumer.close()

client.close()
```

## Step 5: Create a Topic and Subscription

```bash
# Create a persistent topic with 5 partitions
docker exec pulsar bin/pulsar-admin topics create-partitioned-topic \
  persistent://public/default/orders \
  --partitions 5

# Create a subscription
docker exec pulsar bin/pulsar-admin topics create-subscription \
  persistent://public/default/orders \
  --subscription order-processor \
  --messageId earliest
```

## Step 6: Production Configuration

For production, deploy Pulsar in cluster mode using Helm or the Pulsar Operator. For standalone Docker, increase memory limits:

```yaml
services:
  pulsar:
    image: apachepulsar/pulsar:3.2.0
    command: bin/pulsar standalone
    deploy:
      resources:
        limits:
          cpus: '4.0'
          memory: 8G
        reservations:
          cpus: '1.0'
          memory: 2G
    environment:
      - PULSAR_MEM=-Xms2g -Xmx4g -XX:MaxDirectMemorySize=2g
```

## Conclusion

Apache Pulsar in standalone mode provides a complete messaging and streaming platform in a single container. For production workloads requiring high availability and geo-replication, deploy using the Pulsar Kubernetes Operator with separate broker, bookkeeper, and ZooKeeper tiers. The Pulsar admin API at port 8080 provides full topic and tenant management without additional tooling.
