# How to Deploy RabbitMQ via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, RabbitMQ, Message Queues, AMQP, Self-Hosted

Description: Deploy RabbitMQ via Portainer with the management plugin enabled for a feature-rich message broker with web-based administration.

## Introduction

RabbitMQ is a widely-used message broker implementing AMQP, MQTT, and STOMP protocols. It's excellent for task queues, pub/sub messaging, and service decoupling. Deploying via Portainer with the management plugin gives you a complete message broker with a built-in web UI.

## Deploy as a Stack

In Portainer, create a stack named `rabbitmq`:

```yaml
version: "3.8"

services:
  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    container_name: rabbitmq
    hostname: rabbitmq  # Important for cluster stability
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: change_this_password
      RABBITMQ_DEFAULT_VHOST: /
    volumes:
      # Persistent data storage
      - rabbitmq_data:/var/lib/rabbitmq
      # Custom configuration
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      # Definitions for pre-configured exchanges/queues
      - ./definitions.json:/etc/rabbitmq/definitions.json:ro
    ports:
      - "5672:5672"    # AMQP port
      - "15672:15672"  # Management UI
      - "1883:1883"    # MQTT (if enabled)
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  rabbitmq_data:
```

## RabbitMQ Configuration

Create `rabbitmq.conf`:

```ini
# rabbitmq.conf

# Default virtual host

default_vhost = /

# Heartbeat timeout in seconds
heartbeat = 60

# Maximum message size (128MB)
max_message_size = 134217728

# Memory alarm threshold (80% of RAM)
vm_memory_high_watermark.relative = 0.8

# Disk alarm threshold (1GB free)
disk_free_limit.absolute = 1GB

# Enable management plugin
management.load_definitions = /etc/rabbitmq/definitions.json

# Logging
log.console = true
log.console.level = info
log.file = /var/log/rabbitmq/rabbit.log
log.file.level = info
```

## Pre-Configure Exchanges and Queues

Create `definitions.json`:

```json
{
  "vhosts": [
    {"name": "/"},
    {"name": "myapp"}
  ],
  "users": [
    {
      "name": "admin",
      "password_hash": "your_bcrypt_hash",
      "tags": "administrator"
    },
    {
      "name": "app_user",
      "password_hash": "your_bcrypt_hash",
      "tags": ""
    }
  ],
  "permissions": [
    {
      "user": "app_user",
      "vhost": "myapp",
      "configure": ".*",
      "write": ".*",
      "read": ".*"
    }
  ],
  "exchanges": [
    {
      "name": "events",
      "vhost": "myapp",
      "type": "topic",
      "durable": true,
      "auto_delete": false
    }
  ],
  "queues": [
    {
      "name": "email_notifications",
      "vhost": "myapp",
      "durable": true,
      "auto_delete": false,
      "arguments": {
        "x-message-ttl": 86400000,
        "x-max-length": 10000
      }
    },
    {
      "name": "sms_notifications",
      "vhost": "myapp",
      "durable": true,
      "auto_delete": false
    }
  ],
  "bindings": [
    {
      "source": "events",
      "vhost": "myapp",
      "destination": "email_notifications",
      "destination_type": "queue",
      "routing_key": "notification.email.#"
    }
  ]
}
```

## Sending and Receiving Messages

### Python Producer

```python
import pika
import json

connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='localhost',
        virtual_host='myapp',
        credentials=pika.PlainCredentials('app_user', 'password')
    )
)
channel = connection.channel()

# Publish a message
channel.basic_publish(
    exchange='events',
    routing_key='notification.email.welcome',
    body=json.dumps({'to': 'user@example.com', 'subject': 'Welcome!'}),
    properties=pika.BasicProperties(
        delivery_mode=pika.DeliveryMode.Persistent  # Persist to disk
    )
)
print("Message sent!")
connection.close()
```

### Python Consumer

```python
import pika
import json

def process_message(ch, method, properties, body):
    message = json.loads(body)
    print(f"Processing email to: {message['to']}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost', virtual_host='myapp',
        credentials=pika.PlainCredentials('app_user', 'password'))
)
channel = connection.channel()
channel.basic_qos(prefetch_count=1)  # Process one message at a time
channel.basic_consume(queue='email_notifications', on_message_callback=process_message)

print("Waiting for messages...")
channel.start_consuming()
```

## Monitoring RabbitMQ

Access the management UI at `http://<host>:15672` with your admin credentials. Key metrics to watch:

- **Queue depth**: Growing queues indicate consumer lag
- **Message rates**: publish/deliver rates per queue
- **Memory usage**: Should stay below 80% watermark
- **Connections**: Track for connection leaks

```bash
# CLI monitoring
docker exec rabbitmq rabbitmqctl status
docker exec rabbitmq rabbitmqctl list_queues name messages consumers
docker exec rabbitmq rabbitmqctl list_exchanges
```

## Conclusion

RabbitMQ deployed via Portainer provides a production-ready message broker with the management UI included. Pre-configuring exchanges, queues, and bindings via the definitions file means your messaging topology is reproducible and version-controlled. The persistent volume ensures queued messages survive container restarts.
