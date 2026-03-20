# How to Set Up Kafka Connect Workers with IPv4 REST Endpoints

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kafka, Kafka Connect, IPv4, REST API, Configuration, Worker, Integration

Description: Learn how to configure Kafka Connect distributed workers to bind their REST API to a specific IPv4 address for managing connectors over the network.

---

Kafka Connect provides a REST API for managing connectors. By default it listens on `localhost:8083`. Configuring it to bind to a specific IPv4 address allows remote management of connectors in a distributed Connect cluster.

## Worker Configuration (connect-distributed.properties)

```properties
# /etc/kafka/connect-distributed.properties

# --- Kafka cluster connection ---

bootstrap.servers=10.0.0.10:9092,10.0.0.11:9092,10.0.0.12:9092

# --- REST API binding ---
# Bind the REST API to the worker's IPv4 address
rest.host.name=10.0.0.20
rest.port=8083

# The REST API endpoint that this worker advertises to other workers
# Must be reachable from other Connect workers
rest.advertised.host.name=10.0.0.20
rest.advertised.port=8083

# --- Worker identity ---
group.id=connect-cluster-1    # All workers sharing this ID form a cluster

# --- Storage topics (create these before starting workers) ---
config.storage.topic=connect-configs
offset.storage.topic=connect-offsets
status.storage.topic=connect-status

config.storage.replication.factor=3
offset.storage.replication.factor=3
status.storage.replication.factor=3

# --- Converters ---
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false

# --- Plugin path ---
plugin.path=/usr/share/java/kafka-connect-plugins
```

## Starting a Connect Worker

```bash
# Start Kafka Connect in distributed mode
connect-distributed.sh /etc/kafka/connect-distributed.properties &

# Verify the REST API is listening on the IPv4 address
ss -tlnp | grep 8083
curl -s http://10.0.0.20:8083/ | python3 -m json.tool
```

## Managing Connectors via REST API

```bash
# List all connectors
curl -s http://10.0.0.20:8083/connectors | python3 -m json.tool

# Deploy a new connector (example: File Source Connector)
curl -X POST http://10.0.0.20:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "file-source-demo",
    "config": {
      "connector.class": "FileStreamSource",
      "tasks.max": "1",
      "file": "/tmp/test-input.txt",
      "topic": "test-topic"
    }
  }'

# Check connector status
curl -s http://10.0.0.20:8083/connectors/file-source-demo/status | python3 -m json.tool

# Pause a connector
curl -X PUT http://10.0.0.20:8083/connectors/file-source-demo/pause

# Delete a connector
curl -X DELETE http://10.0.0.20:8083/connectors/file-source-demo
```

## Multi-Worker Cluster Setup

For a 3-worker Connect cluster:

```bash
# Each worker on its own IPv4 address, same group.id
# Worker 1: rest.host.name=10.0.0.20, rest.advertised.host.name=10.0.0.20
# Worker 2: rest.host.name=10.0.0.21, rest.advertised.host.name=10.0.0.21
# Worker 3: rest.host.name=10.0.0.22, rest.advertised.host.name=10.0.0.22

# You can send REST requests to ANY worker in the cluster
# Workers coordinate and route to the correct owner
curl -s http://10.0.0.21:8083/connectors  # Works from any worker
```

## Key Takeaways

- `rest.host.name` binds the REST API to a specific IPv4 address; set `rest.advertised.host.name` to the same IP for correct cluster coordination.
- All workers in the same Connect cluster must share the same `group.id` and storage topic names.
- REST requests can be sent to any worker; the cluster handles routing internally.
- Protect the REST API with a reverse proxy (Nginx/HAProxy) and TLS for production deployments.
